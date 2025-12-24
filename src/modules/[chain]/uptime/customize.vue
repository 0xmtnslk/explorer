<script lang="ts" setup>
import { ref, onMounted, computed, watchEffect } from 'vue';
import { fromHex, toBase64 } from '@cosmjs/encoding';
import { Icon } from '@iconify/vue';
import { useFormatter, useStakingStore, useBaseStore, useBlockchain, useDashboard } from '@/stores';
import UptimeBar from '@/components/UptimeBar.vue';
import type { Block, Commit } from '@/types';
import { consensusPubkeyToHexAddress, valconsToBase64 } from '@/libs';
import type { SigningInfo } from '@/types/slashing';
import { CosmosRestClient } from '@/libs/client';

const props = defineProps(['chain']);

const stakingStore = useStakingStore();
const format = useFormatter();
const chainStore = useBlockchain();
const dashboard = useDashboard();
const local = ref(
  JSON.parse(localStorage.getItem('uptime-validators') || '{}') as Record<string, { name: string; address: string }[]>
);
const signingInfo = ref({} as Record<string, SigningInfo[]>);
const selected = ref([] as string[]);
const selectChain = ref(chainStore.chainName);
const validators = ref(stakingStore.validators);
const keyword = ref('');
const showModal = ref(false);

function loadSigningInfo(chainName: string) {
  const chain = dashboard.chains[chainName];
  if (chain && chain.endpoints.rest) {
    const client = CosmosRestClient.newDefault(chain.endpoints.rest[0].address);
    client.getSlashingSigningInfos().then((resp) => {
      signingInfo.value[chainName] = resp.info;
    });
  }
}

if (local.value)
  Object.keys(local.value).map((chainName) => {
    loadSigningInfo(chainName);
  });

function initial() {
  const vals = local.value[selectChain.value];
  if (vals) {
    selected.value = vals.map((x) => x.address);
  }
}

const filterValidators = computed(() => {
  if (keyword.value) {
    return validators.value.filter((x) => x.description.moniker.toLowerCase().indexOf(keyword.value.toLowerCase()) > -1);
  }
  return validators.value;
});

const list = computed(() => {
  const list = [] as any[];
  if (local.value)
    Object.keys(local.value).map((chainName) => {
      const vals = local.value[chainName];
      const info = signingInfo.value[chainName];
      if (vals && info) {
        vals.forEach((v) => {
          const sigingInfo = info.find((x) => valconsToBase64(x.address) === v.address);
          list.push({
            chainName,
            v,
            sigingInfo,
          });
        });
      }
    });
  return list;
});

function add() {
  if (!signingInfo.value[selectChain.value]) {
    loadSigningInfo(selectChain.value);
  }
  const newList = [] as { name: string; address: string }[];
  selected.value.forEach((x) => {
    const validator = validators.value.find((v) => consensusPubkeyToHexAddress(v.consensus_pubkey) === x);
    if (validator)
      newList.push({
        name: validator.description.moniker || x,
        address: x,
      });
  });
  if (!local.value) local.value = {};
  local.value[selectChain.value] = newList;

  localStorage.setItem('uptime-validators', JSON.stringify(local.value));
  showModal.value = false;
}

function changeChain() {
  validators.value = [];
  const endpoint = dashboard.chains[selectChain.value].endpoints.rest?.at(0)?.address;
  if (!endpoint) return;

  const client = CosmosRestClient.newDefault(endpoint);
  client.getStakingValidators('BOND_STATUS_BONDED').then((x) => {
    validators.value = x.validators;
  });

  const vals = local.value[selectChain.value];
  if (vals) {
    selected.value = vals.map((x) => x.address);
  } else {
    selected.value = [];
  }
}

function openModal() {
  showModal.value = true;
  initial();
}

function color(v: string) {
  if (v) {
    const n = Number(v);
    if (n < 10) return 'bg-green-100 dark:bg-green-900/30 text-green-600 dark:text-green-400';
    if (n > 1000) return 'bg-red-100 dark:bg-red-900/30 text-red-600 dark:text-red-400';
    if (n > 0) return 'bg-yellow-100 dark:bg-yellow-900/30 text-yellow-600 dark:text-yellow-400';
  }
  return 'bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-400';
}
</script>

<template>
  <div class="space-y-6">
    <!-- Hero Section -->
    <div class="relative overflow-hidden rounded-2xl bg-gradient-to-r from-purple-500 via-pink-500 to-rose-500 p-6 md:p-8">
      <div class="absolute inset-0 bg-black/10"></div>
      <div class="absolute -top-24 -right-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      <div class="absolute -bottom-24 -left-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      
      <div class="relative z-10">
        <div class="flex items-center gap-3 mb-2">
          <div class="w-12 h-12 rounded-xl bg-white/20 backdrop-blur flex items-center justify-center">
            <Icon icon="mdi:account-star" class="text-2xl text-white" />
          </div>
          <div>
            <h1 class="text-2xl md:text-3xl font-bold text-white">{{ $t('uptime.my_validators') }}</h1>
            <p class="text-white/80 text-sm">{{ $t('uptime.add_validators_monitor') }}</p>
          </div>
        </div>
        
        <!-- Stats -->
        <div class="flex flex-wrap gap-4 mt-4">
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Tracked Validators</div>
            <div class="text-white font-bold text-lg">{{ list.length }}</div>
          </div>
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Networks</div>
            <div class="text-white font-bold text-lg">{{ Object.keys(local || {}).length }}</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Back Link + Add Button -->
    <div class="flex items-center justify-between">
      <RouterLink :to="`/${chain}/uptime`" class="flex items-center gap-2 text-gray-600 dark:text-gray-400 hover:text-primary transition-colors">
        <Icon icon="mdi:arrow-left" class="text-lg" />
        <span>Back to Uptime</span>
      </RouterLink>
      
      <button 
        @click="openModal"
        class="flex items-center gap-2 px-4 py-2.5 bg-gradient-to-r from-purple-500 to-pink-500 text-white rounded-xl font-medium hover:shadow-lg hover:shadow-purple-500/25 transition-all"
      >
        <Icon icon="mdi:plus" class="text-lg" />
        {{ $t('uptime.add_validators') }}
      </button>
    </div>

    <!-- Validators Table -->
    <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 overflow-hidden">
      <div v-if="list.length === 0" class="text-center py-16">
        <div class="w-20 h-20 mx-auto mb-4 rounded-full bg-gray-100 dark:bg-gray-700 flex items-center justify-center">
          <Icon icon="mdi:server-off" class="text-4xl text-gray-400" />
        </div>
        <h3 class="text-lg font-medium text-gray-800 dark:text-white mb-2">No Validators Added</h3>
        <p class="text-gray-500 dark:text-gray-400 mb-6">Start tracking your validators by adding them to your watchlist</p>
        <button 
          @click="openModal"
          class="inline-flex items-center gap-2 px-6 py-3 bg-gradient-to-r from-purple-500 to-pink-500 text-white rounded-xl font-medium hover:shadow-lg hover:shadow-purple-500/25 transition-all"
        >
          <Icon icon="mdi:plus" />
          {{ $t('uptime.add_validators') }}
        </button>
      </div>

      <div v-else class="overflow-x-auto">
        <table class="w-full">
          <thead>
            <tr class="bg-gradient-to-r from-gray-50 to-gray-100 dark:from-gray-700 dark:to-gray-800">
              <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">#</th>
              <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">Blockchain</th>
              <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('account.validator') }}</th>
              <th class="text-right px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.signed_blocks') }}</th>
              <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.last_jailed_time') }}</th>
              <th class="text-center px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.tombstoned') }}</th>
              <th class="text-right px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.missing_blocks') }}</th>
              <th class="text-right px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider"></th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-100 dark:divide-gray-700">
            <tr v-for="(v, i) in list" :key="i" class="hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-colors">
              <td class="px-6 py-4">
                <span class="w-8 h-8 rounded-full bg-gradient-to-r from-purple-500 to-pink-500 text-white text-sm flex items-center justify-center font-medium">
                  {{ i + 1 }}
                </span>
              </td>
              <td class="px-6 py-4">
                <div class="flex items-center gap-2">
                  <img 
                    v-if="dashboard.chains[v.chainName]?.logo" 
                    :src="dashboard.chains[v.chainName].logo" 
                    class="w-6 h-6 rounded-full"
                    :alt="v.chainName"
                  />
                  <span class="text-sm font-medium text-gray-800 dark:text-white capitalize">{{ v.chainName }}</span>
                </div>
              </td>
              <td class="px-6 py-4">
                <span class="text-sm font-medium text-gray-800 dark:text-white">{{ v.v.name }}</span>
              </td>
              <td class="px-6 py-4 text-right">
                <span v-if="v.sigingInfo" class="text-sm text-gray-600 dark:text-gray-400">
                  {{ Number(v.sigingInfo.index_offset) - Number(v.sigingInfo.start_height) }}
                </span>
                <span v-else class="text-sm text-gray-400">-</span>
              </td>
              <td class="px-6 py-4">
                <div v-if="v.sigingInfo && !v.sigingInfo?.jailed_until.startsWith('1970')" class="text-sm text-gray-600 dark:text-gray-400">
                  {{ format.toDay(v.sigingInfo.jailed_until, 'from') }}
                </div>
                <span v-else class="text-sm text-gray-400">-</span>
              </td>
              <td class="px-6 py-4 text-center">
                <span 
                  v-if="v.sigingInfo?.tombstoned === true"
                  class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-red-100 dark:bg-red-900/30 text-red-700 dark:text-red-400"
                >
                  Yes
                </span>
                <span 
                  v-else
                  class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 dark:bg-green-900/30 text-green-700 dark:text-green-400"
                >
                  No
                </span>
              </td>
              <td class="px-6 py-4 text-right">
                <span 
                  v-if="v.sigingInfo" 
                  class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-bold"
                  :class="color(v.sigingInfo?.missed_blocks_counter)"
                >
                  {{ v.sigingInfo?.missed_blocks_counter }}
                </span>
              </td>
              <td class="px-6 py-4 text-right">
                <RouterLink 
                  :to="`/${v.chainName}/uptime`" 
                  class="inline-flex items-center gap-1 px-3 py-1.5 bg-primary/10 text-primary rounded-lg text-sm font-medium hover:bg-primary/20 transition-all"
                >
                  <Icon icon="mdi:eye" />
                  View
                </RouterLink>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Add Validator Modal -->
    <div v-if="showModal" class="fixed inset-0 z-50 flex items-center justify-center p-4">
      <div class="fixed inset-0 bg-black/50 backdrop-blur-sm" @click="showModal = false"></div>
      
      <div class="relative bg-white dark:bg-gray-800 rounded-2xl shadow-2xl w-full max-w-lg max-h-[90vh] overflow-hidden">
        <!-- Modal Header -->
        <div class="flex items-center justify-between px-6 py-4 border-b border-gray-200 dark:border-gray-700">
          <div class="flex items-center gap-3">
            <div class="w-10 h-10 rounded-xl bg-gradient-to-r from-purple-500 to-pink-500 flex items-center justify-center">
              <Icon icon="mdi:plus" class="text-xl text-white" />
            </div>
            <h3 class="text-lg font-bold text-gray-800 dark:text-white">{{ $t('uptime.add_validators') }}</h3>
          </div>
          <button @click="showModal = false" class="w-8 h-8 rounded-full bg-gray-100 dark:bg-gray-700 flex items-center justify-center hover:bg-gray-200 dark:hover:bg-gray-600 transition-colors">
            <Icon icon="mdi:close" class="text-gray-500" />
          </button>
        </div>

        <!-- Modal Body -->
        <div class="p-6 space-y-4">
          <!-- Chain & Search -->
          <div class="flex gap-3">
            <select 
              v-model="selectChain" 
              @change="changeChain"
              class="flex-shrink-0 px-4 py-2.5 bg-gray-100 dark:bg-gray-700 border-0 rounded-xl text-sm font-medium text-gray-700 dark:text-gray-300 focus:ring-2 focus:ring-primary/50 capitalize"
            >
              <option v-for="v in dashboard.chains" :key="v.chainName" :value="v.chainName">
                {{ v.chainName }}
              </option>
            </select>
            <div class="relative flex-1">
              <Icon icon="mdi:magnify" class="absolute left-3 top-1/2 -translate-y-1/2 text-gray-400" />
              <input 
                v-model="keyword" 
                type="text" 
                placeholder="Search validators..." 
                class="w-full pl-10 pr-4 py-2.5 bg-gray-100 dark:bg-gray-700 border-0 rounded-xl text-sm text-gray-700 dark:text-gray-300 focus:ring-2 focus:ring-primary/50 placeholder-gray-400"
              />
            </div>
          </div>

          <!-- Validators List -->
          <div class="max-h-72 overflow-y-auto rounded-xl border border-gray-200 dark:border-gray-700">
            <div 
              v-for="(v, i) in filterValidators" 
              :key="v.operator_address"
              class="flex items-center justify-between px-4 py-3 hover:bg-gray-50 dark:hover:bg-gray-700/50 border-b border-gray-100 dark:border-gray-700 last:border-b-0 cursor-pointer transition-colors"
              @click="selected.includes(consensusPubkeyToHexAddress(v.consensus_pubkey)) 
                ? selected = selected.filter(s => s !== consensusPubkeyToHexAddress(v.consensus_pubkey))
                : selected.push(consensusPubkeyToHexAddress(v.consensus_pubkey))"
            >
              <div class="flex items-center gap-3">
                <span class="w-6 h-6 rounded-full bg-gray-100 dark:bg-gray-700 text-xs flex items-center justify-center text-gray-600 dark:text-gray-400">
                  {{ i + 1 }}
                </span>
                <span class="text-sm font-medium text-gray-800 dark:text-white">{{ v.description.moniker }}</span>
              </div>
              <div 
                class="w-5 h-5 rounded border-2 flex items-center justify-center transition-all"
                :class="selected.includes(consensusPubkeyToHexAddress(v.consensus_pubkey)) 
                  ? 'bg-primary border-primary' 
                  : 'border-gray-300 dark:border-gray-600'"
              >
                <Icon 
                  v-if="selected.includes(consensusPubkeyToHexAddress(v.consensus_pubkey))" 
                  icon="mdi:check" 
                  class="text-white text-sm" 
                />
              </div>
            </div>
            <div v-if="filterValidators.length === 0" class="px-4 py-8 text-center text-gray-500">
              No validators found
            </div>
          </div>

          <div class="text-sm text-gray-500 dark:text-gray-400">
            {{ selected.length }} validator(s) selected
          </div>
        </div>

        <!-- Modal Footer -->
        <div class="flex items-center justify-end gap-3 px-6 py-4 border-t border-gray-200 dark:border-gray-700 bg-gray-50 dark:bg-gray-700/50">
          <button 
            @click="showModal = false"
            class="px-4 py-2 text-sm font-medium text-gray-700 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-600 rounded-lg transition-colors"
          >
            Cancel
          </button>
          <button 
            @click="add"
            class="px-6 py-2 bg-gradient-to-r from-purple-500 to-pink-500 text-white rounded-lg text-sm font-medium hover:shadow-lg hover:shadow-purple-500/25 transition-all"
          >
            {{ $t('uptime.add') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
