<script lang="ts" setup>
import { useBlockchain, useFormatter, useStakingStore, useTxDialog } from '@/stores';
import DynamicComponent from '@/components/dynamic/DynamicComponent.vue';
import { computed, ref } from '@vue/reactivity';
import { onMounted } from 'vue';
import { Icon } from '@iconify/vue';

import type { AuthAccount, Delegation, TxResponse, DelegatorRewards, UnbondingResponses } from '@/types';
import type { Coin } from '@cosmjs/amino';
import Countdown from '@/components/Countdown.vue';
import { fromBase64 } from '@cosmjs/encoding';

const props = defineProps(['address', 'chain']);

const blockchain = useBlockchain();
const stakingStore = useStakingStore();
const dialog = useTxDialog();
const format = useFormatter();
const account = ref({} as AuthAccount);
const txs = ref({} as TxResponse[]);
const delegations = ref([] as Delegation[]);
const rewards = ref({} as DelegatorRewards);
const balances = ref([] as Coin[]);
const recentReceived = ref([] as TxResponse[]);
const unbonding = ref([] as UnbondingResponses[]);
const unbondingTotal = ref(0);
const showAllTxs = ref(false);
const showAllReceived = ref(false);
const activeTab = ref('assets');
const isLoading = ref(true);

onMounted(() => {
  if (props.address) {
    loadAccount(props.address);
  }
});

const totalAmountByCategory = computed(() => {
  let sumDel = 0;
  delegations.value?.forEach((x) => {
    sumDel += Number(x.balance.amount);
  });
  let sumRew = 0;
  rewards.value?.total?.forEach((x) => {
    sumRew += Number(x.amount);
  });
  let sumBal = 0;
  balances.value?.forEach((x) => {
    sumBal += Number(x.amount);
  });
  let sumUn = 0;
  unbonding.value?.forEach((x) => {
    x.entries?.forEach((y) => {
      sumUn += Number(y.balance);
    });
  });
  return { balance: sumBal, delegation: sumDel, reward: sumRew, unbonding: sumUn };
});

const totalAmount = computed(() => {
  const cat = totalAmountByCategory.value;
  return cat.balance + cat.delegation + cat.reward + cat.unbonding;
});

const totalValue = computed(() => {
  let value = 0;
  delegations.value?.forEach((x) => {
    value += format.tokenValueNumber(x.balance);
  });
  rewards.value?.total?.forEach((x) => {
    value += format.tokenValueNumber(x);
  });
  balances.value?.forEach((x) => {
    value += format.tokenValueNumber(x);
  });
  unbonding.value?.forEach((x) => {
    x.entries?.forEach((y) => {
      value += format.tokenValueNumber({ amount: y.balance, denom: stakingStore.params.bond_denom });
    });
  });
  return value;
});

const displayedTxs = computed(() => {
  const allTxs = txs.value || [];
  const reversedTxs = [...allTxs].reverse();
  return showAllTxs.value ? reversedTxs : reversedTxs.slice(0, 5);
});

const displayedReceived = computed(() => {
  const allReceived = recentReceived.value || [];
  return showAllReceived.value ? allReceived : allReceived.slice(0, 5);
});

function loadAccount(address: string) {
  isLoading.value = true;
  
  Promise.all([
    blockchain.rpc.getAuthAccount(address).then((x) => {
      account.value = x.account;
    }).catch(() => {}),
    blockchain.rpc.getTxsBySender(address).then((x) => {
      txs.value = x.tx_responses || [];
    }).catch(() => { txs.value = []; }),
    blockchain.rpc.getDistributionDelegatorRewards(address).then((x) => {
      rewards.value = x;
    }).catch(() => {}),
    blockchain.rpc.getStakingDelegations(address).then((x) => {
      delegations.value = x.delegation_responses || [];
    }).catch(() => { delegations.value = []; }),
    blockchain.rpc.getBankBalances(address).then((x) => {
      balances.value = x.balances || [];
    }).catch(() => { balances.value = []; }),
    blockchain.rpc.getStakingDelegatorUnbonding(address).then((x) => {
      unbonding.value = x.unbonding_responses || [];
      x.unbonding_responses?.forEach((y) => {
        y.entries.forEach((z) => {
          unbondingTotal.value += Number(z.balance);
        });
      });
    }).catch(() => { unbonding.value = []; }),
  ]).finally(() => {
    isLoading.value = false;
  });

  const receivedQuery = `?&pagination.reverse=true&events=coin_received.receiver='${address}'&pagination.limit=20`;
  blockchain.rpc.getTxs(receivedQuery, {}).then((x) => {
    recentReceived.value = x.tx_responses || [];
  }).catch(() => { recentReceived.value = []; });
}

function updateEvent() {
  loadAccount(props.address);
}

function mapAmount(events: { type: string; attributes: { key: string; value: string }[] }[]) {
  if (!events) return [];
  return events
    .find((x) => x.type === 'coin_received')
    ?.attributes.filter((x) => x.key === 'YW1vdW50' || x.key === `amount`)
    .map((x) => (x.key === 'amount' ? x.value : String.fromCharCode(...fromBase64(x.value))));
}

const copyAddress = async () => {
  try {
    await navigator.clipboard.writeText(props.address);
  } catch (err) {}
};

const shortenAddress = (addr: string) => {
  if (!addr) return '';
  return addr.slice(0, 12) + '...' + addr.slice(-8);
};
</script>

<template>
  <div v-if="isLoading" class="flex items-center justify-center min-h-[400px]">
    <div class="text-center">
      <div class="w-12 h-12 border-4 border-primary border-t-transparent rounded-full animate-spin mx-auto mb-4"></div>
      <p class="text-gray-500 dark:text-gray-400">Loading account data...</p>
    </div>
  </div>
  
  <div v-else-if="address" class="space-y-6">
    <!-- Hero Section - Wallet Overview -->
    <div class="relative overflow-hidden rounded-2xl bg-gradient-to-br from-gray-50 via-white to-gray-100 dark:from-slate-900 dark:via-slate-800 dark:to-slate-900 border border-gray-200/50 dark:border-gray-700/50">
      <div class="absolute inset-0 bg-[radial-gradient(ellipse_at_top_right,_var(--tw-gradient-stops))] from-primary/10 dark:from-primary/20 via-transparent to-transparent"></div>
      <div class="absolute top-0 right-0 w-64 h-64 bg-gradient-to-br from-primary/5 to-emerald-500/5 rounded-full blur-3xl"></div>
      
      <div class="relative p-6 lg:p-8">
        <div class="flex flex-col lg:flex-row gap-6 items-start lg:items-center justify-between">
          <!-- Left: Address Info -->
          <div class="flex items-center gap-4">
            <div class="w-16 h-16 rounded-2xl bg-gradient-to-br from-primary to-emerald-500 flex items-center justify-center shadow-lg">
              <Icon icon="mdi:wallet" class="text-3xl text-white" />
            </div>
            <div>
              <h1 class="text-xl lg:text-2xl font-bold text-gray-900 dark:text-white mb-1">Wallet Account</h1>
              <div class="flex items-center gap-2">
                <span class="text-sm text-gray-500 dark:text-gray-400 font-mono">{{ shortenAddress(address) }}</span>
                <button @click="copyAddress" class="p-1 hover:bg-gray-200 dark:hover:bg-gray-700 rounded transition-colors">
                  <Icon icon="mdi:content-copy" class="text-gray-400 hover:text-primary" />
                </button>
              </div>
            </div>
          </div>

          <!-- Right: Total Value & Actions -->
          <div class="flex flex-col items-end gap-3">
            <div class="text-right">
              <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide">Total Value</div>
              <div class="text-3xl lg:text-4xl font-bold bg-gradient-to-r from-primary to-emerald-500 bg-clip-text text-transparent">
                ${{ format.formatNumber(totalValue, '0,0.00') }}
              </div>
            </div>
            <div class="flex gap-2">
              <button class="px-4 py-2 rounded-xl bg-primary text-white text-sm font-medium hover:bg-primary/90 transition-colors flex items-center gap-1.5" @click="dialog.open('send', {}, updateEvent)">
                <Icon icon="mdi:send" />
                Send
              </button>
              <button class="px-4 py-2 rounded-xl bg-gray-200 dark:bg-gray-700 text-gray-700 dark:text-gray-300 text-sm font-medium hover:bg-gray-300 dark:hover:bg-gray-600 transition-colors flex items-center gap-1.5" @click="dialog.open('transfer', { chain_name: blockchain.current?.prettyName }, updateEvent)">
                <Icon icon="mdi:swap-horizontal" />
                IBC Transfer
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Asset Summary Cards -->
    <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4">
        <div class="flex items-center gap-3 mb-3">
          <div class="w-10 h-10 rounded-xl bg-blue-500/10 flex items-center justify-center">
            <Icon icon="mdi:wallet" class="text-xl text-blue-500" />
          </div>
          <span class="text-sm text-gray-500 dark:text-gray-400">Available</span>
        </div>
        <div class="text-lg font-bold text-gray-900 dark:text-white">
          {{ format.formatToken2({ amount: String(totalAmountByCategory.balance), denom: stakingStore.params.bond_denom }) }}
        </div>
        <div class="text-xs text-gray-400 mt-1">${{ format.formatNumber(format.tokenValueNumber({ amount: String(totalAmountByCategory.balance), denom: stakingStore.params.bond_denom }), '0,0.00') }}</div>
      </div>

      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4">
        <div class="flex items-center gap-3 mb-3">
          <div class="w-10 h-10 rounded-xl bg-amber-500/10 flex items-center justify-center">
            <Icon icon="mdi:lock" class="text-xl text-amber-500" />
          </div>
          <span class="text-sm text-gray-500 dark:text-gray-400">Staked</span>
        </div>
        <div class="text-lg font-bold text-gray-900 dark:text-white">
          {{ format.formatToken2({ amount: String(totalAmountByCategory.delegation), denom: stakingStore.params.bond_denom }) }}
        </div>
        <div class="text-xs text-gray-400 mt-1">${{ format.formatNumber(format.tokenValueNumber({ amount: String(totalAmountByCategory.delegation), denom: stakingStore.params.bond_denom }), '0,0.00') }}</div>
      </div>

      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4">
        <div class="flex items-center gap-3 mb-3">
          <div class="w-10 h-10 rounded-xl bg-emerald-500/10 flex items-center justify-center">
            <Icon icon="mdi:gift" class="text-xl text-emerald-500" />
          </div>
          <span class="text-sm text-gray-500 dark:text-gray-400">Rewards</span>
        </div>
        <div class="text-lg font-bold text-emerald-500">
          {{ format.formatToken2({ amount: String(Math.floor(totalAmountByCategory.reward)), denom: stakingStore.params.bond_denom }) }}
        </div>
        <div class="text-xs text-gray-400 mt-1">${{ format.formatNumber(format.tokenValueNumber({ amount: String(totalAmountByCategory.reward), denom: stakingStore.params.bond_denom }), '0,0.00') }}</div>
      </div>

      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4">
        <div class="flex items-center gap-3 mb-3">
          <div class="w-10 h-10 rounded-xl bg-red-500/10 flex items-center justify-center">
            <Icon icon="mdi:clock-outline" class="text-xl text-red-500" />
          </div>
          <span class="text-sm text-gray-500 dark:text-gray-400">Unbonding</span>
        </div>
        <div class="text-lg font-bold text-gray-900 dark:text-white">
          {{ format.formatToken2({ amount: String(unbondingTotal), denom: stakingStore.params.bond_denom }) }}
        </div>
        <div class="text-xs text-gray-400 mt-1">${{ format.formatNumber(format.tokenValueNumber({ amount: String(unbondingTotal), denom: stakingStore.params.bond_denom }), '0,0.00') }}</div>
      </div>
    </div>

    <!-- Assets Detail -->
    <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50">
      <div class="p-6 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-3">
            <div class="p-2.5 rounded-xl bg-gradient-to-br from-primary to-emerald-500">
              <Icon icon="mdi:chart-pie" class="text-xl text-white" />
            </div>
            <h2 class="text-lg font-semibold text-gray-900 dark:text-white">Asset Breakdown</h2>
          </div>
        </div>
      </div>
      <div class="p-6 space-y-3">
        <div v-for="(balanceItem, index) in balances" :key="'bal-'+index" class="flex items-center p-3 rounded-xl bg-gray-50 dark:bg-gray-700/50 hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors">
          <div class="w-10 h-10 rounded-xl bg-blue-500/10 flex items-center justify-center mr-4">
            <Icon icon="mdi:circle-multiple" class="text-xl text-blue-500" />
          </div>
          <div class="flex-1">
            <div class="text-sm font-semibold text-gray-900 dark:text-white">{{ format.formatToken(balanceItem) }}</div>
            <div class="text-xs text-gray-500">Available Balance</div>
          </div>
          <div class="text-right">
            <div class="text-sm font-semibold text-primary">${{ format.tokenValue(balanceItem) }}</div>
            <div class="text-xs text-gray-400">{{ format.calculatePercent(balanceItem.amount, totalAmount) }}</div>
          </div>
        </div>

        <div v-for="(delegationItem, index) in delegations" :key="'del-'+index" class="flex items-center p-3 rounded-xl bg-gray-50 dark:bg-gray-700/50 hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors">
          <div class="w-10 h-10 rounded-xl bg-amber-500/10 flex items-center justify-center mr-4">
            <Icon icon="mdi:lock" class="text-xl text-amber-500" />
          </div>
          <div class="flex-1">
            <div class="text-sm font-semibold text-gray-900 dark:text-white">{{ format.formatToken(delegationItem?.balance) }}</div>
            <div class="text-xs text-gray-500">
              <RouterLink :to="`/${chain}/staking/${delegationItem.delegation.validator_address}`" class="text-primary hover:underline">
                {{ format.validatorFromBech32(delegationItem.delegation.validator_address) || 'Validator' }}
              </RouterLink>
            </div>
          </div>
          <div class="text-right">
            <div class="text-sm font-semibold text-primary">${{ format.tokenValue(delegationItem?.balance) }}</div>
            <div class="text-xs text-gray-400">{{ format.calculatePercent(delegationItem?.balance?.amount, totalAmount) }}</div>
          </div>
        </div>

        <div v-for="(rewardItem, index) in rewards.total" :key="'rew-'+index" class="flex items-center p-3 rounded-xl bg-gray-50 dark:bg-gray-700/50 hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors">
          <div class="w-10 h-10 rounded-xl bg-emerald-500/10 flex items-center justify-center mr-4">
            <Icon icon="mdi:gift" class="text-xl text-emerald-500" />
          </div>
          <div class="flex-1">
            <div class="text-sm font-semibold text-emerald-500">{{ format.formatToken(rewardItem) }}</div>
            <div class="text-xs text-gray-500">Pending Rewards</div>
          </div>
          <div class="text-right">
            <div class="text-sm font-semibold text-primary">${{ format.tokenValue(rewardItem) }}</div>
            <div class="text-xs text-gray-400">{{ format.calculatePercent(rewardItem.amount, totalAmount) }}</div>
          </div>
        </div>

        <div v-if="unbondingTotal > 0" class="flex items-center p-3 rounded-xl bg-gray-50 dark:bg-gray-700/50 hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors">
          <div class="w-10 h-10 rounded-xl bg-red-500/10 flex items-center justify-center mr-4">
            <Icon icon="mdi:clock-outline" class="text-xl text-red-500" />
          </div>
          <div class="flex-1">
            <div class="text-sm font-semibold text-gray-900 dark:text-white">{{ format.formatToken({ amount: String(unbondingTotal), denom: stakingStore.params.bond_denom }) }}</div>
            <div class="text-xs text-gray-500">Unbonding</div>
          </div>
          <div class="text-right">
            <div class="text-sm font-semibold text-primary">${{ format.tokenValue({ amount: String(unbondingTotal), denom: stakingStore.params.bond_denom }) }}</div>
            <div class="text-xs text-gray-400">{{ format.calculatePercent(unbondingTotal, totalAmount) }}</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Delegations -->
    <div v-if="delegations.length > 0" class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50">
      <div class="p-6 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-3">
            <div class="p-2.5 rounded-xl bg-gradient-to-br from-amber-500 to-orange-500">
              <Icon icon="mdi:account-group" class="text-xl text-white" />
            </div>
            <h2 class="text-lg font-semibold text-gray-900 dark:text-white">Staking Delegations</h2>
            <span class="px-2 py-0.5 rounded-full text-xs font-medium bg-amber-100 dark:bg-amber-900/30 text-amber-600 dark:text-amber-400">{{ delegations.length }}</span>
          </div>
          <div class="flex gap-2">
            <button class="px-3 py-1.5 rounded-lg bg-primary text-white text-xs font-medium hover:bg-primary/90 transition-colors" @click="dialog.open('delegate', {}, updateEvent)">Delegate</button>
            <button class="px-3 py-1.5 rounded-lg bg-emerald-500 text-white text-xs font-medium hover:bg-emerald-600 transition-colors" @click="dialog.open('withdraw', {}, updateEvent)">Claim All</button>
          </div>
        </div>
      </div>
      <div class="overflow-x-auto">
        <table class="w-full">
          <thead class="bg-gray-50 dark:bg-gray-700/50">
            <tr>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Validator</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Delegation</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Rewards</th>
              <th class="text-right text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Actions</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-200 dark:divide-gray-700">
            <tr v-for="(v, index) in delegations" :key="index" class="hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-colors">
              <td class="px-6 py-4">
                <RouterLink :to="`/${chain}/staking/${v.delegation.validator_address}`" class="text-sm font-medium text-primary hover:underline">
                  {{ format.validatorFromBech32(v.delegation.validator_address) || v.delegation.validator_address.slice(0, 20) + '...' }}
                </RouterLink>
              </td>
              <td class="px-6 py-4">
                <div class="text-sm font-medium text-gray-900 dark:text-white">{{ format.formatToken(v.balance, true, '0,0.[0000]') }}</div>
                <div class="text-xs text-gray-500">${{ format.tokenValue(v.balance) }}</div>
              </td>
              <td class="px-6 py-4">
                <div class="text-sm font-medium text-emerald-500">
                  {{ format.formatTokens(rewards?.rewards?.find((x) => x.validator_address === v.delegation.validator_address)?.reward) }}
                </div>
              </td>
              <td class="px-6 py-4">
                <div class="flex justify-end gap-1">
                  <button class="px-2 py-1 rounded text-xs font-medium bg-primary/10 text-primary hover:bg-primary/20 transition-colors" @click="dialog.open('delegate', { validator_address: v.delegation.validator_address }, updateEvent)">Delegate</button>
                  <button class="px-2 py-1 rounded text-xs font-medium bg-amber-500/10 text-amber-600 hover:bg-amber-500/20 transition-colors" @click="dialog.open('redelegate', { validator_address: v.delegation.validator_address }, updateEvent)">Redelegate</button>
                  <button class="px-2 py-1 rounded text-xs font-medium bg-red-500/10 text-red-600 hover:bg-red-500/20 transition-colors" @click="dialog.open('unbond', { validator_address: v.delegation.validator_address }, updateEvent)">Unbond</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Unbonding Delegations -->
    <div v-if="unbonding && unbonding.length > 0" class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50">
      <div class="p-6 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center gap-3">
          <div class="p-2.5 rounded-xl bg-gradient-to-br from-red-500 to-pink-500">
            <Icon icon="mdi:clock-alert" class="text-xl text-white" />
          </div>
          <h2 class="text-lg font-semibold text-gray-900 dark:text-white">Unbonding Delegations</h2>
        </div>
      </div>
      <div class="overflow-x-auto">
        <table class="w-full">
          <thead class="bg-gray-50 dark:bg-gray-700/50">
            <tr>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Validator</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Amount</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Completion</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-200 dark:divide-gray-700">
            <template v-for="(v, index) in unbonding" :key="index">
              <tr v-for="entry in v.entries" :key="entry.creation_height" class="hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-colors">
                <td class="px-6 py-4">
                  <RouterLink :to="`/${chain}/staking/${v.validator_address}`" class="text-sm font-medium text-primary hover:underline">
                    {{ format.validatorFromBech32(v.validator_address) || v.validator_address.slice(0, 20) + '...' }}
                  </RouterLink>
                </td>
                <td class="px-6 py-4">
                  <div class="text-sm font-medium text-gray-900 dark:text-white">{{ format.formatToken({ amount: entry.balance, denom: stakingStore.params.bond_denom }, true, '0,0.[00]') }}</div>
                </td>
                <td class="px-6 py-4">
                  <div class="flex items-center gap-2">
                    <Icon icon="mdi:timer-sand" class="text-amber-500" />
                    <Countdown :time="new Date(entry.completion_time).getTime() - new Date().getTime()" />
                  </div>
                </td>
              </tr>
            </template>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Transactions -->
    <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50">
      <div class="p-6 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-3">
            <div class="p-2.5 rounded-xl bg-gradient-to-br from-violet-500 to-purple-500">
              <Icon icon="mdi:swap-horizontal-bold" class="text-xl text-white" />
            </div>
            <h2 class="text-lg font-semibold text-gray-900 dark:text-white">Transaction History</h2>
            <span class="px-2 py-0.5 rounded-full text-xs font-medium bg-violet-100 dark:bg-violet-900/30 text-violet-600 dark:text-violet-400">{{ txs.length || 0 }}</span>
          </div>
        </div>
      </div>
      <div class="overflow-x-auto">
        <table class="w-full">
          <thead class="bg-gray-50 dark:bg-gray-700/50">
            <tr>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Hash</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Type</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Height</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Time</th>
              <th class="text-center text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Status</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-200 dark:divide-gray-700">
            <tr v-if="!txs || txs.length === 0">
              <td colspan="5" class="px-6 py-8 text-center text-gray-500 dark:text-gray-400">No transactions found</td>
            </tr>
            <tr v-for="(v, index) in displayedTxs" :key="index" class="hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-colors">
              <td class="px-6 py-4">
                <RouterLink :to="`/${chain}/tx/${v.txhash}`" class="text-sm font-mono text-primary hover:underline">
                  {{ v.txhash?.slice(0, 8) }}...{{ v.txhash?.slice(-6) }}
                </RouterLink>
              </td>
              <td class="px-6 py-4">
                <span class="px-2 py-1 rounded-full text-xs font-medium bg-gray-100 dark:bg-gray-700 text-gray-700 dark:text-gray-300">
                  {{ format.messages(v.tx.body.messages) }}
                </span>
              </td>
              <td class="px-6 py-4">
                <RouterLink :to="`/${chain}/block/${v.height}`" class="text-sm text-primary hover:underline">
                  {{ v.height }}
                </RouterLink>
              </td>
              <td class="px-6 py-4">
                <div class="text-sm text-gray-900 dark:text-white">{{ format.toDay(v.timestamp, 'from') }}</div>
                <div class="text-xs text-gray-500">{{ format.toLocaleDate(v.timestamp) }}</div>
              </td>
              <td class="px-6 py-4 text-center">
                <span v-if="v.code === 0" class="inline-flex items-center gap-1 px-2 py-1 rounded-full text-xs font-medium bg-emerald-100 dark:bg-emerald-900/30 text-emerald-600 dark:text-emerald-400">
                  <Icon icon="mdi:check-circle" />
                  Success
                </span>
                <span v-else class="inline-flex items-center gap-1 px-2 py-1 rounded-full text-xs font-medium bg-red-100 dark:bg-red-900/30 text-red-600 dark:text-red-400">
                  <Icon icon="mdi:close-circle" />
                  Failed
                </span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <div v-if="txs && txs.length > 5" class="p-4 border-t border-gray-200/50 dark:border-gray-700/50 text-center">
        <button @click="showAllTxs = !showAllTxs" class="px-4 py-2 rounded-lg bg-gray-100 dark:bg-gray-700 text-gray-700 dark:text-gray-300 text-sm font-medium hover:bg-gray-200 dark:hover:bg-gray-600 transition-colors">
          {{ showAllTxs ? 'Show Less' : `View All ${txs.length} Transactions` }}
        </button>
      </div>
    </div>

    <!-- Recent Received -->
    <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50">
      <div class="p-6 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center gap-3">
          <div class="p-2.5 rounded-xl bg-gradient-to-br from-emerald-500 to-teal-500">
            <Icon icon="mdi:arrow-down-bold" class="text-xl text-white" />
          </div>
          <h2 class="text-lg font-semibold text-gray-900 dark:text-white">Recent Received</h2>
          <span class="px-2 py-0.5 rounded-full text-xs font-medium bg-emerald-100 dark:bg-emerald-900/30 text-emerald-600 dark:text-emerald-400">{{ recentReceived.length || 0 }}</span>
        </div>
      </div>
      <div class="overflow-x-auto">
        <table class="w-full">
          <thead class="bg-gray-50 dark:bg-gray-700/50">
            <tr>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Hash</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Amount</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Height</th>
              <th class="text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Time</th>
              <th class="text-center text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider px-6 py-3">Status</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-200 dark:divide-gray-700">
            <tr v-if="recentReceived.length === 0">
              <td colspan="5" class="px-6 py-8 text-center text-gray-500 dark:text-gray-400">No received transactions found</td>
            </tr>
            <tr v-for="(v, index) in displayedReceived" :key="index" class="hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-colors">
              <td class="px-6 py-4">
                <RouterLink :to="`/${chain}/tx/${v.txhash}`" class="text-sm font-mono text-primary hover:underline">
                  {{ v.txhash?.slice(0, 8) }}...{{ v.txhash?.slice(-6) }}
                </RouterLink>
              </td>
              <td class="px-6 py-4">
                <span class="text-sm font-medium text-emerald-500">{{ mapAmount(v.events)?.join(', ') || '-' }}</span>
              </td>
              <td class="px-6 py-4">
                <RouterLink :to="`/${chain}/block/${v.height}`" class="text-sm text-primary hover:underline">
                  {{ v.height }}
                </RouterLink>
              </td>
              <td class="px-6 py-4">
                <div class="text-sm text-gray-900 dark:text-white">{{ format.toDay(v.timestamp, 'from') }}</div>
                <div class="text-xs text-gray-500">{{ format.toLocaleDate(v.timestamp) }}</div>
              </td>
              <td class="px-6 py-4 text-center">
                <span v-if="v.code === 0" class="inline-flex items-center gap-1 px-2 py-1 rounded-full text-xs font-medium bg-emerald-100 dark:bg-emerald-900/30 text-emerald-600 dark:text-emerald-400">
                  <Icon icon="mdi:check-circle" />
                  Success
                </span>
                <span v-else class="inline-flex items-center gap-1 px-2 py-1 rounded-full text-xs font-medium bg-red-100 dark:bg-red-900/30 text-red-600 dark:text-red-400">
                  <Icon icon="mdi:close-circle" />
                  Failed
                </span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <div v-if="recentReceived && recentReceived.length > 5" class="p-4 border-t border-gray-200/50 dark:border-gray-700/50 text-center">
        <button @click="showAllReceived = !showAllReceived" class="px-4 py-2 rounded-lg bg-gray-100 dark:bg-gray-700 text-gray-700 dark:text-gray-300 text-sm font-medium hover:bg-gray-200 dark:hover:bg-gray-600 transition-colors">
          {{ showAllReceived ? 'Show Less' : `View All ${recentReceived.length} Received` }}
        </button>
      </div>
    </div>

    <!-- Account Details -->
    <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50">
      <div class="p-6 border-b border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center gap-3">
          <div class="p-2.5 rounded-xl bg-gradient-to-br from-gray-500 to-slate-500">
            <Icon icon="mdi:code-json" class="text-xl text-white" />
          </div>
          <h2 class="text-lg font-semibold text-gray-900 dark:text-white">Account Details</h2>
        </div>
      </div>
      <div class="p-6">
        <DynamicComponent :value="account" />
      </div>
    </div>
  </div>
  
  <div v-else class="flex items-center justify-center min-h-[400px]">
    <div class="text-center">
      <Icon icon="mdi:alert-circle" class="text-5xl text-gray-400 mb-4" />
      <p class="text-gray-500 dark:text-gray-400">Account not found or loading...</p>
    </div>
  </div>
</template>
