<script lang="ts" setup>
import { useBaseStore, useBlockchain, useFormatter, useMintStore, useStakingStore, useTxDialog } from '@/stores';
import { computed } from '@vue/reactivity';
import { onMounted, ref, onUnmounted } from 'vue';
import { Icon } from '@iconify/vue';
import type { Key, SlashingParam, Validator, SigningInfo } from '@/types';
import { formatSeconds } from '@/libs/utils';
import { consensusPubkeyToHexAddress, valconsToBase64 } from '@/libs';
import { fromHex, toBase64 } from '@cosmjs/encoding';

const staking = useStakingStore();
const base = useBaseStore();
const format = useFormatter();
const dialog = useTxDialog();
const chainStore = useBlockchain();
const mintStore = useMintStore();

const cache = JSON.parse(localStorage.getItem('avatars') || '{}');
const avatars = ref(cache || {});
const latest = ref({} as Record<string, number>);
const yesterday = ref({} as Record<string, number>);
const tab = ref('active');
const unbondList = ref([] as Validator[]);
const slashingParam = ref({} as SlashingParam);
const signingInfo = ref({} as Record<string, SigningInfo>);
const searchActive = ref('');
const searchInactive = ref('');

onMounted(() => {
  staking.fetchUnbondingValdiators().then((res) => {
    unbondList.value = res.concat(unbondList.value);
  });
  staking.fetchInacitveValdiators().then((res) => {
    unbondList.value = unbondList.value.concat(res);
  });
  chainStore.rpc.getSlashingParams().then((res) => {
    slashingParam.value = res.params;
  });
  updateSigningInfo();
});

function updateSigningInfo() {
  chainStore.rpc.getSlashingSigningInfos().then((x) => {
    x.info?.forEach((i) => {
      signingInfo.value[valconsToBase64(i.address)] = i;
    });
  });
}

const getValidatorUptime = (validator: Validator) => {
  try {
    const hex = consensusPubkeyToHexAddress(validator.consensus_pubkey);
    const base64 = toBase64(fromHex(hex));
    const signing = signingInfo.value[base64];
    const window = Number(slashingParam.value.signed_blocks_window || 0);
    if (signing && window > 0) {
      return ((window - Number(signing.missed_blocks_counter)) / window) * 100;
    }
  } catch (e) {}
  return null;
};

function isFeatured(endpoints: string[], who?: { website?: string; moniker: string }) {
  if (!endpoints || !who) return false;
  return (
    endpoints.findIndex(
      (x) =>
        (who.website && who.website?.substring(0, who.website?.lastIndexOf('.')).endsWith(x)) ||
        who?.moniker?.toLowerCase().search(x.toLowerCase()) > -1
    ) > -1
  );
}

const providerValidator = computed(() => {
  const endpoint = chainStore.current?.endpoints?.rest?.map((x) => x.provider);
  if (endpoint) {
    endpoint.push('ping');
    const found = staking.validators.find((x) => isFeatured(endpoint.filter(Boolean) as string[], x.description));
    if (found) {
      return {
        v: found,
        logo: logo(found.description.identity),
        uptime: getValidatorUptime(found)
      };
    }
  }
  return null;
});

const activeValidators = computed(() => {
  let validators = staking.validators.map((x) => ({
    v: x,
    logo: logo(x.description.identity),
    uptime: getValidatorUptime(x)
  }));
  
  if (searchActive.value) {
    validators = validators.filter(item => 
      item.v.description?.moniker?.toLowerCase().includes(searchActive.value.toLowerCase())
    );
  }
  
  return validators;
});

const inactiveValidators = computed(() => {
  let validators = unbondList.value.map((x) => ({
    v: x,
    logo: logo(x.description.identity),
    uptime: null as number | null
  }));
  
  if (searchInactive.value) {
    validators = validators.filter(item => 
      item.v.description?.moniker?.toLowerCase().includes(searchInactive.value.toLowerCase())
    );
  }
  
  return validators;
});

const fetchAvatar = (identity: string) => {
  return new Promise<void>((resolve) => {
    staking
      .keybase(identity)
      .then((d) => {
        if (Array.isArray(d.them) && d.them.length > 0) {
          const uri = String(d.them[0]?.pictures?.primary?.url).replace(
            'https://s3.amazonaws.com/keybase_processed_uploads/',
            ''
          );
          avatars.value[identity] = uri;
          resolve();
        } else throw new Error(`failed to fetch avatar for ${identity}`);
      })
      .catch(() => {
        resolve();
      });
  });
};

const loadAvatar = (identity: string) => {
  fetchAvatar(identity).then(() => {
    localStorage.setItem('avatars', JSON.stringify(avatars.value));
  });
};

const loadAvatars = () => {
  const promises = staking.validators.map((validator) => {
    const identity = validator.description?.identity;
    if (identity && !avatars.value[identity]) {
      return fetchAvatar(identity);
    } else {
      return Promise.resolve();
    }
  });
  Promise.all(promises).then(() => localStorage.setItem('avatars', JSON.stringify(avatars.value)));
};

const logo = (identity?: string) => {
  if (!identity || !avatars.value[identity]) return '';
  const url = avatars.value[identity] || '';
  return url.startsWith('http') ? url : `https://s3.amazonaws.com/keybase_processed_uploads/${url}`;
};

const formatTokens = (tokens: string) => {
  return format.formatTokenAmount({
    amount: tokens,
    denom: staking.params.bond_denom,
  });
};

const formatCommission = (rate: string) => {
  return format.percent(rate);
};

const getUptimeColor = (uptime: number | null) => {
  if (uptime === null) return 'text-gray-400';
  if (uptime >= 99) return 'text-emerald-500';
  if (uptime >= 95) return 'text-yellow-500';
  return 'text-red-500';
};

const getUptimeBgColor = (uptime: number | null) => {
  if (uptime === null) return 'bg-gray-500/10';
  if (uptime >= 99) return 'bg-emerald-500/10';
  if (uptime >= 95) return 'bg-yellow-500/10';
  return 'bg-red-500/10';
};

loadAvatars();
</script>

<template>
  <div class="space-y-6">
    <!-- Header Stats - Modern Design -->
    <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
      <div class="relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-emerald-400/50 transition-all">
        <div class="absolute -top-6 -right-6 w-20 h-20 bg-gradient-to-br from-emerald-500/20 to-green-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10">
          <div class="flex items-center gap-2 mb-2">
            <div class="p-2 rounded-xl bg-gradient-to-br from-emerald-500 to-green-600">
              <Icon icon="mdi:trending-up" class="text-xl text-white" />
            </div>
          </div>
          <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ format.percent(mintStore.inflation) }}</div>
          <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">{{ $t('staking.inflation') }}</div>
        </div>
      </div>

      <div class="relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-blue-400/50 transition-all">
        <div class="absolute -top-6 -right-6 w-20 h-20 bg-gradient-to-br from-blue-500/20 to-cyan-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10">
          <div class="flex items-center gap-2 mb-2">
            <div class="p-2 rounded-xl bg-gradient-to-br from-blue-500 to-cyan-600">
              <Icon icon="mdi:lock-open-outline" class="text-xl text-white" />
            </div>
          </div>
          <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ formatSeconds(staking.params?.unbonding_time) }}</div>
          <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">{{ $t('staking.unbonding_time') }}</div>
        </div>
      </div>

      <div class="relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-red-400/50 transition-all">
        <div class="absolute -top-6 -right-6 w-20 h-20 bg-gradient-to-br from-red-500/20 to-orange-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10">
          <div class="flex items-center gap-2 mb-2">
            <div class="p-2 rounded-xl bg-gradient-to-br from-red-500 to-orange-600">
              <Icon icon="mdi:alert-octagon-outline" class="text-xl text-white" />
            </div>
          </div>
          <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ format.percent(slashingParam.slash_fraction_double_sign) }}</div>
          <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">{{ $t('staking.double_sign_slashing') }}</div>
        </div>
      </div>

      <div class="relative overflow-hidden rounded-2xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 group hover:border-amber-400/50 transition-all">
        <div class="absolute -top-6 -right-6 w-20 h-20 bg-gradient-to-br from-amber-500/20 to-yellow-500/20 rounded-full blur-xl group-hover:scale-150 transition-transform"></div>
        <div class="relative z-10">
          <div class="flex items-center gap-2 mb-2">
            <div class="p-2 rounded-xl bg-gradient-to-br from-amber-500 to-yellow-600">
              <Icon icon="mdi:pause" class="text-xl text-white" />
            </div>
          </div>
          <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ format.percent(slashingParam.slash_fraction_downtime) }}</div>
          <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mt-1">{{ $t('staking.downtime_slashing') }}</div>
        </div>
      </div>
    </div>

    <!-- Provider/Featured Validator Section -->
    <div v-if="providerValidator" class="relative overflow-hidden rounded-2xl bg-gradient-to-br from-primary via-primary/90 to-emerald-600 p-6 text-white">
      <div class="absolute top-0 right-0 w-40 h-40 bg-white/10 rounded-full -translate-y-1/2 translate-x-1/2"></div>
      <div class="absolute bottom-0 left-0 w-32 h-32 bg-white/5 rounded-full translate-y-1/2 -translate-x-1/2"></div>
      <div class="relative z-10">
        <div class="flex items-center gap-2 mb-4">
          <Icon icon="mdi:star" class="text-yellow-300 text-xl" />
          <span class="text-sm font-medium uppercase tracking-wider opacity-90">Recommended Validator</span>
        </div>
        
        <div class="flex flex-col md:flex-row md:items-center justify-between gap-6">
          <RouterLink 
            :to="{ name: 'chain-staking-validator', params: { validator: providerValidator.v.operator_address } }"
            class="flex items-center gap-4 hover:opacity-80 transition-opacity cursor-pointer"
          >
            <div class="w-16 h-16 rounded-2xl bg-white/20 backdrop-blur-sm flex items-center justify-center overflow-hidden">
              <img v-if="providerValidator.logo" :src="providerValidator.logo" class="w-14 h-14 rounded-xl object-cover" />
              <Icon v-else icon="mdi:account" class="text-4xl text-white/80" />
            </div>
            <div>
              <h3 class="text-2xl font-bold hover:underline">{{ providerValidator.v.description?.moniker }}</h3>
              <p class="text-white/70 text-sm mt-1">{{ providerValidator.v.description?.website }}</p>
            </div>
          </RouterLink>
          
          <div class="flex flex-wrap items-center gap-6">
            <div class="text-center">
              <div class="text-2xl font-bold">{{ formatTokens(providerValidator.v.tokens) }}</div>
              <div class="text-xs text-white/70 uppercase">Staked</div>
            </div>
            <div class="text-center">
              <div class="text-2xl font-bold">{{ formatCommission(providerValidator.v.commission?.commission_rates?.rate) }}</div>
              <div class="text-xs text-white/70 uppercase">Commission</div>
            </div>
            <div class="text-center">
              <div class="text-2xl font-bold">{{ providerValidator.uptime !== null ? providerValidator.uptime.toFixed(2) + '%' : 'N/A' }}</div>
              <div class="text-xs text-white/70 uppercase">Uptime</div>
            </div>
            <button
              class="px-6 py-3 rounded-xl bg-white text-primary font-semibold hover:bg-white/90 transition-all shadow-lg"
              @click="dialog.open('delegate', { validator_address: providerValidator.v.operator_address })"
            >
              <Icon icon="mdi:hand-coin" class="inline mr-2" />
              Delegate
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Tabs Section -->
    <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/80 backdrop-blur-xl border border-gray-200 dark:border-gray-700/50 shadow-sm">
      <!-- Tab Headers -->
      <div class="flex items-center justify-between px-6 py-4 border-b border-gray-100 dark:border-gray-700/50">
        <div class="flex items-center gap-2">
          <button 
            class="px-5 py-2.5 rounded-xl font-semibold text-sm transition-all"
            :class="tab === 'active' 
              ? 'bg-gradient-to-r from-primary to-emerald-600 text-white shadow-lg shadow-primary/20' 
              : 'bg-gray-100 dark:bg-gray-700/50 text-gray-600 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-600/50'"
            @click="tab = 'active'"
          >
            <Icon icon="mdi:check-circle" class="inline mr-1.5" />
            Active ({{ staking.validators.length }})
          </button>
          <button 
            class="px-5 py-2.5 rounded-xl font-semibold text-sm transition-all"
            :class="tab === 'inactive' 
              ? 'bg-gradient-to-r from-gray-500 to-gray-600 text-white shadow-lg shadow-gray-500/20' 
              : 'bg-gray-100 dark:bg-gray-700/50 text-gray-600 dark:text-gray-300 hover:bg-gray-200 dark:hover:bg-gray-600/50'"
            @click="tab = 'inactive'"
          >
            <Icon icon="mdi:close-circle" class="inline mr-1.5" />
            Inactive ({{ unbondList.length }})
          </button>
        </div>
        <div class="text-sm text-gray-500 dark:text-gray-400">
          Max: {{ staking.params.max_validators }} validators
        </div>
      </div>

      <!-- Active Validators -->
      <div v-if="tab === 'active'" class="p-6">
        <!-- Search -->
        <div class="mb-6">
          <div class="relative">
            <Icon icon="mdi:magnify" class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400 text-xl" />
            <input
              v-model="searchActive"
              type="text"
              placeholder="Search active validators..."
              class="w-full pl-12 pr-4 py-3 rounded-xl bg-gray-50 dark:bg-gray-700/50 border border-gray-200 dark:border-gray-600 focus:border-primary focus:ring-2 focus:ring-primary/20 outline-none transition-all text-gray-900 dark:text-white"
            />
          </div>
        </div>

        <!-- Validator Cards Grid -->
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
          <div
            v-for="({ v, logo, uptime }, i) in activeValidators"
            :key="v.operator_address"
            class="relative overflow-hidden rounded-2xl bg-gray-50 dark:bg-gray-700/30 border border-gray-200/50 dark:border-gray-600/30 p-5 hover:border-primary/50 hover:shadow-lg hover:shadow-primary/5 transition-all group"
          >
            <!-- Rank Badge -->
            <div class="absolute top-3 right-3">
              <span class="px-2.5 py-1 rounded-full text-xs font-bold bg-primary/10 text-primary">
                #{{ i + 1 }}
              </span>
            </div>

            <!-- Validator Info -->
            <div class="flex items-start gap-3 mb-4">
              <div class="w-12 h-12 rounded-xl bg-gray-200 dark:bg-gray-600 flex items-center justify-center overflow-hidden flex-shrink-0">
                <img 
                  v-if="logo" 
                  :src="logo" 
                  class="w-full h-full object-cover"
                  @error="(e) => { const identity = v.description?.identity; if (identity) loadAvatar(identity); }"
                />
                <Icon v-else icon="mdi:account" class="text-2xl text-gray-400" />
              </div>
              <div class="flex-1 min-w-0">
                <RouterLink
                  :to="{ name: 'chain-staking-validator', params: { validator: v.operator_address } }"
                  class="font-semibold text-gray-900 dark:text-white hover:text-primary transition-colors block truncate"
                >
                  {{ v.description?.moniker }}
                </RouterLink>
                <p class="text-xs text-gray-500 dark:text-gray-400 truncate mt-0.5">{{ v.description?.website || 'No website' }}</p>
              </div>
            </div>

            <!-- Stats Grid -->
            <div class="grid grid-cols-3 gap-2 mb-4">
              <div class="text-center p-2 rounded-lg bg-white dark:bg-gray-800/50">
                <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Staked</div>
                <div class="text-sm font-semibold text-gray-900 dark:text-white truncate">{{ formatTokens(v.tokens) }}</div>
              </div>
              <div class="text-center p-2 rounded-lg bg-white dark:bg-gray-800/50">
                <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Commission</div>
                <div class="text-sm font-semibold text-gray-900 dark:text-white">{{ formatCommission(v.commission?.commission_rates?.rate) }}</div>
              </div>
              <div class="text-center p-2 rounded-lg" :class="getUptimeBgColor(uptime)">
                <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Uptime</div>
                <div class="text-sm font-semibold" :class="getUptimeColor(uptime)">
                  {{ uptime !== null ? uptime.toFixed(1) + '%' : 'N/A' }}
                </div>
              </div>
            </div>

            <!-- Delegate Button -->
            <button
              class="w-full py-2.5 rounded-xl bg-gradient-to-r from-primary to-emerald-600 text-white font-semibold text-sm hover:from-primary/90 hover:to-emerald-600/90 transition-all shadow-md shadow-primary/20 flex items-center justify-center gap-2"
              @click="dialog.open('delegate', { validator_address: v.operator_address })"
            >
              <Icon icon="mdi:hand-coin" />
              Delegate
            </button>
          </div>
        </div>

        <div v-if="activeValidators.length === 0" class="text-center py-12 text-gray-500 dark:text-gray-400">
          <Icon icon="mdi:account-search" class="text-5xl mb-3 opacity-50" />
          <p>No validators found matching "{{ searchActive }}"</p>
        </div>
      </div>

      <!-- Inactive Validators -->
      <div v-if="tab === 'inactive'" class="p-6">
        <!-- Search -->
        <div class="mb-6">
          <div class="relative">
            <Icon icon="mdi:magnify" class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400 text-xl" />
            <input
              v-model="searchInactive"
              type="text"
              placeholder="Search inactive validators..."
              class="w-full pl-12 pr-4 py-3 rounded-xl bg-gray-50 dark:bg-gray-700/50 border border-gray-200 dark:border-gray-600 focus:border-primary focus:ring-2 focus:ring-primary/20 outline-none transition-all text-gray-900 dark:text-white"
            />
          </div>
        </div>

        <!-- Inactive Validator Cards Grid -->
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
          <div
            v-for="({ v, logo }, i) in inactiveValidators"
            :key="v.operator_address"
            class="relative overflow-hidden rounded-2xl bg-gray-100 dark:bg-gray-700/20 border border-gray-200/50 dark:border-gray-600/20 p-5 opacity-80 hover:opacity-100 transition-all"
          >
            <!-- Status Badge -->
            <div class="absolute top-3 right-3">
              <span 
                class="px-2.5 py-1 rounded-full text-xs font-bold"
                :class="v.jailed ? 'bg-red-500/10 text-red-500' : 'bg-gray-500/10 text-gray-500'"
              >
                {{ v.jailed ? 'Jailed' : 'Inactive' }}
              </span>
            </div>

            <!-- Validator Info -->
            <div class="flex items-start gap-3 mb-4">
              <div class="w-12 h-12 rounded-xl bg-gray-200 dark:bg-gray-600 flex items-center justify-center overflow-hidden flex-shrink-0 grayscale">
                <img 
                  v-if="logo" 
                  :src="logo" 
                  class="w-full h-full object-cover"
                />
                <Icon v-else icon="mdi:account" class="text-2xl text-gray-400" />
              </div>
              <div class="flex-1 min-w-0">
                <RouterLink
                  :to="{ name: 'chain-staking-validator', params: { validator: v.operator_address } }"
                  class="font-semibold text-gray-700 dark:text-gray-300 hover:text-primary transition-colors block truncate"
                >
                  {{ v.description?.moniker }}
                </RouterLink>
                <p class="text-xs text-gray-500 dark:text-gray-400 truncate mt-0.5">{{ v.description?.website || 'No website' }}</p>
              </div>
            </div>

            <!-- Stats Grid -->
            <div class="grid grid-cols-3 gap-2 mb-4">
              <div class="text-center p-2 rounded-lg bg-white/50 dark:bg-gray-800/30">
                <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Staked</div>
                <div class="text-sm font-semibold text-gray-700 dark:text-gray-300 truncate">{{ formatTokens(v.tokens) }}</div>
              </div>
              <div class="text-center p-2 rounded-lg bg-white/50 dark:bg-gray-800/30">
                <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Commission</div>
                <div class="text-sm font-semibold text-gray-700 dark:text-gray-300">{{ formatCommission(v.commission?.commission_rates?.rate) }}</div>
              </div>
              <div class="text-center p-2 rounded-lg bg-gray-300/30 dark:bg-gray-600/30">
                <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Uptime</div>
                <div class="text-sm font-semibold text-gray-400">N/A</div>
              </div>
            </div>

            <!-- Info Text -->
            <div class="text-center text-xs text-gray-500 dark:text-gray-400 py-2">
              <Icon icon="mdi:information" class="inline mr-1" />
              {{ v.jailed ? 'This validator is currently jailed' : 'This validator is not in the active set' }}
            </div>
          </div>
        </div>

        <div v-if="inactiveValidators.length === 0" class="text-center py-12 text-gray-500 dark:text-gray-400">
          <Icon icon="mdi:account-search" class="text-5xl mb-3 opacity-50" />
          <p v-if="searchInactive">No validators found matching "{{ searchInactive }}"</p>
          <p v-else>No inactive validators</p>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
</style>
