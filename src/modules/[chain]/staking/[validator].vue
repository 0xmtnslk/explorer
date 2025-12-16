<script setup lang="ts">
import { parseCoins } from '@cosmjs/stargate';
import {
  useBankStore,
  useBaseStore,
  useBlockchain,
  useDistributionStore,
  useFormatter,
  useMintStore,
  useStakingStore,
  useTxDialog,
} from '@/stores';
import { onMounted, computed, ref } from 'vue';
import { Icon } from '@iconify/vue';
import { consensusPubkeyToHexAddress, operatorAddressToAccount, pubKeyToValcons, valconsToBase64 } from '@/libs';
import {
  PageRequest,
  type Coin,
  type Delegation,
  type PaginatedDelegations,
  type PaginatedTxs,
  type Validator,
  type SlashingParam,
  type SigningInfo,
} from '@/types';
import PaginationBar from '@/components/PaginationBar.vue';
import { fromBase64, toBase64, fromHex } from '@cosmjs/encoding';
import { stringToUint8Array, uint8ArrayToString } from '@/libs/utils';

const props = defineProps(['validator', 'chain']);

const staking = useStakingStore();
const blockchain = useBlockchain();
const baseStore = useBaseStore();
const format = useFormatter();
const dialog = useTxDialog();
const page = new PageRequest();

const validator: string = props.validator;

const v = ref({} as Validator);
const cache = JSON.parse(localStorage.getItem('avatars') || '{}');
const avatars = ref(cache || {});
const identity = ref('');
const rewards = ref([] as Coin[] | undefined);
const commission = ref([] as Coin[] | undefined);
const delegations = ref({} as PaginatedDelegations);
const addresses = ref({} as {
  account: string;
  operAddress: string;
  hex: string;
  valCons: string;
});
const selfBonded = ref({} as Delegation);
const slashingParam = ref({} as SlashingParam);
const signingInfo = ref({} as Record<string, SigningInfo>);
const uptime = ref<number | null>(null);
const showAllTxs = ref(false);
const showAllEvents = ref(false);
const liveBlocks = ref<{ height: string; signed: boolean }[]>([]);
const validatorBase64 = ref('');
const loadingBlocks = ref(true);

addresses.value.account = operatorAddressToAccount(validator);

staking.fetchValidatorDelegation(validator, addresses.value.account).then((x) => {
  if (x) {
    selfBonded.value = x.delegation_response;
  }
});

const txs = ref({} as PaginatedTxs);

blockchain.rpc.getTxsBySender(addresses.value.account).then((x) => {
  txs.value = x;
});

blockchain.rpc.getSlashingParams().then((res) => {
  slashingParam.value = res.params;
});

blockchain.rpc.getSlashingSigningInfos().then((x) => {
  x.info?.forEach((i) => {
    signingInfo.value[valconsToBase64(i.address)] = i;
  });
});

const calculateUptime = () => {
  if (v.value.consensus_pubkey) {
    try {
      const hex = consensusPubkeyToHexAddress(v.value.consensus_pubkey);
      const base64 = toBase64(fromHex(hex));
      validatorBase64.value = base64;
      const signing = signingInfo.value[base64];
      const window = Number(slashingParam.value.signed_blocks_window || 0);
      if (signing && window > 0) {
        uptime.value = ((window - Number(signing.missed_blocks_counter)) / window) * 100;
      }
      initLiveBlocks();
    } catch (e) {}
  }
};

const initLiveBlocks = async () => {
  if (!validatorBase64.value) return;
  loadingBlocks.value = true;
  
  const latestHeight = baseStore.latest?.block?.header?.height;
  if (!latestHeight) {
    loadingBlocks.value = false;
    return;
  }
  
  const blocks: { height: string; signed: boolean }[] = [];
  const startHeight = Math.max(1, Number(latestHeight) - 99);
  
  for (let h = startHeight; h <= Number(latestHeight); h++) {
    try {
      const block = baseStore.recents?.find((b: any) => Number(b.block?.header?.height) === h);
      if (block) {
        const sig = block.block?.last_commit?.signatures?.find((s: any) => s.validator_address === validatorBase64.value);
        blocks.push({
          height: String(h),
          signed: !!sig && sig.block_id_flag === 'BLOCK_ID_FLAG_COMMIT'
        });
      } else {
        const fetched = await blockchain.rpc.getBlockByHeight(h);
        if (fetched?.block) {
          const sig = fetched.block?.last_commit?.signatures?.find((s: any) => s.validator_address === validatorBase64.value);
          blocks.push({
            height: String(h),
            signed: !!sig && sig.block_id_flag === 'BLOCK_ID_FLAG_COMMIT'
          });
        }
      }
    } catch (e) {
      blocks.push({ height: String(h), signed: true });
    }
  }
  
  liveBlocks.value = blocks;
  loadingBlocks.value = false;
};

baseStore.$subscribe((_, state) => {
  if (validatorBase64.value && state.recents?.length > 0) {
    const latestBlock = state.recents[state.recents.length - 1];
    if (latestBlock?.block?.header?.height) {
      const exists = liveBlocks.value.find(b => b.height === latestBlock.block.header.height);
      if (!exists) {
        const sig = latestBlock.block?.last_commit?.signatures?.find((s: any) => s.validator_address === validatorBase64.value);
        liveBlocks.value.push({
          height: latestBlock.block.header.height,
          signed: !!sig && sig.block_id_flag === 'BLOCK_ID_FLAG_COMMIT'
        });
        if (liveBlocks.value.length > 100) liveBlocks.value.shift();
      }
    }
  }
});

const apr = computed(() => {
  const rate = Number(v.value.commission?.commission_rates.rate || 0);
  const inflation = useMintStore().inflation;
  const communityTax = Number(useDistributionStore().params.community_tax);
  const bondedRatio = Number(staking.pool.bonded_tokens) / Number(useBankStore().supply.amount);
  return format.percent(((1 - communityTax) * (1 - rate) * Number(inflation)) / bondedRatio);
});

const selfRate = computed(() => {
  if (selfBonded.value.balance?.amount) {
    return format.calculatePercent(selfBonded.value.balance.amount, v.value.tokens);
  }
  return '-';
});

const commissionRate = computed(() => Number(v.value.commission?.commission_rates?.rate || 0) * 100);
const maxRate = computed(() => Number(v.value.commission?.commission_rates?.max_rate || 0) * 100);
const maxChangeRate = computed(() => Number(v.value.commission?.commission_rates?.max_change_rate || 0) * 100);

const logo = (identity?: string) => {
  if (!identity) return '';
  const url = avatars.value[identity] || '';
  return url.startsWith('http') ? url : `https://s3.amazonaws.com/keybase_processed_uploads/${url}`;
};

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
        } else throw new Error(`failed to fetch avatar for ${identity}.`);
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

onMounted(() => {
  if (validator) {
    staking.fetchValidator(validator).then((res) => {
      v.value = res.validator;
      identity.value = res.validator?.description?.identity || '';
      if (identity.value && !avatars.value[identity.value]) loadAvatar(identity.value);
      addresses.value.hex = consensusPubkeyToHexAddress(v.value.consensus_pubkey);
      addresses.value.valCons = pubKeyToValcons(v.value.consensus_pubkey, blockchain.current?.bech32ConsensusPrefix || '');
      setTimeout(calculateUptime, 1000);
    });
    blockchain.rpc.getDistributionValidatorOutstandingRewards(validator).then((res) => {
      rewards.value = res.rewards?.rewards?.sort((a, b) => Number(b.amount) - Number(a.amount));
      res.rewards?.rewards?.forEach((x) => {
        if (x.denom.startsWith('ibc/')) format.fetchDenomTrace(x.denom);
      });
    });
    blockchain.rpc.getDistributionValidatorCommission(validator).then((res) => {
      commission.value = res.commission?.commission?.sort((a, b) => Number(b.amount) - Number(a.amount));
      res.commission?.commission?.forEach((x) => {
        if (x.denom.startsWith('ibc/')) format.fetchDenomTrace(x.denom);
      });
    });
  }
});

let showCopyToast = ref(0);
const copyToClipboard = async (text: string) => {
  if (!text) return;
  try {
    await navigator.clipboard.writeText(text);
    showCopyToast.value = 1;
    setTimeout(() => { showCopyToast.value = 0; }, 2000);
  } catch (err) {
    showCopyToast.value = 2;
    setTimeout(() => { showCopyToast.value = 0; }, 2000);
  }
};

const events = ref({} as PaginatedTxs);

enum EventType {
  Delegate = 'delegate',
  Unbond = 'unbond',
}

const selectedEventType = ref(EventType.Delegate);

function loadPowerEvents(p: number, type: EventType) {
  selectedEventType.value = type;
  page.setPage(p);
  page.setPageSize(5);
  blockchain.rpc
    .getTxs("?order_by=2&events={type}.validator='{validator}'", { type: selectedEventType.value, validator }, page)
    .then((res) => {
      events.value = res;
    });
}

function pagePowerEvents(page: number) {
  loadPowerEvents(page, selectedEventType.value);
}

pagePowerEvents(1);

function mapEvents(events: { type: string; attributes: { key: string; value: string }[] }[]) {
  const attributes = events
    .filter((x) => x.type === selectedEventType.value)
    .filter((x) => x.attributes.findIndex((attr) => attr.value === validator || attr.value === toBase64(stringToUint8Array(validator))) > -1)
    .map((x) => {
      const output = {} as { [key: string]: string };
      if (x.attributes.findIndex((a) => a.key === `amount`) > -1) {
        x.attributes.forEach((attr) => { output[attr.key] = attr.value; });
      } else {
        x.attributes.forEach((attr) => {
          output[uint8ArrayToString(fromBase64(attr.key))] = uint8ArrayToString(fromBase64(attr.value));
        });
      }
      return output;
    });
  const coinsAsString = attributes.map((x: any) => x.amount).join(',');
  const coins = parseCoins(coinsAsString);
  return coins.map((coin) => format.formatToken(coin)).join(', ');
}

function mapDelegators(messages: any[]) {
  if (!messages) return [];
  return Array.from(new Set(messages.map((x) => x.delegator_address || x.grantee)));
}

const displayedTxs = computed(() => {
  const allTxs = txs.value.tx_responses || [];
  const reversedTxs = [...allTxs].reverse();
  return showAllTxs.value ? reversedTxs : reversedTxs.slice(0, 5);
});

const displayedEvents = computed(() => {
  const allEvents = events.value.tx_responses || [];
  return showAllEvents.value ? allEvents : allEvents.slice(0, 5);
});

const getUptimeColor = (value: number | null) => {
  if (value === null) return 'text-gray-400';
  if (value >= 99) return 'text-emerald-500';
  if (value >= 95) return 'text-yellow-500';
  return 'text-red-500';
};
</script>

<template>
  <div class="space-y-6">
    <!-- Hero Section - Validator Info -->
    <div class="relative overflow-hidden rounded-2xl bg-gradient-to-br from-gray-50 via-white to-gray-100 dark:from-slate-900 dark:via-slate-800 dark:to-slate-900 border border-gray-200/50 dark:border-gray-700/50">
      <div class="absolute inset-0 bg-[radial-gradient(ellipse_at_top_right,_var(--tw-gradient-stops))] from-primary/10 dark:from-primary/20 via-transparent to-transparent"></div>
      <div class="absolute top-0 right-0 w-64 h-64 bg-gradient-to-br from-primary/5 to-emerald-500/5 rounded-full blur-3xl"></div>
      
      <div class="relative p-6 lg:p-8">
        <div class="flex flex-col lg:flex-row gap-6">
          <!-- Left: Avatar & Basic Info -->
          <div class="flex items-start gap-5">
            <div class="w-24 h-24 rounded-2xl bg-gradient-to-br from-primary/20 to-emerald-500/20 flex items-center justify-center overflow-hidden flex-shrink-0 border-2 border-white dark:border-gray-700 shadow-lg">
              <img v-if="identity && avatars[identity]" v-lazy="logo(identity)" class="w-full h-full object-cover" @error="() => loadAvatar(identity)" />
              <Icon v-else icon="mdi:account" class="text-5xl text-gray-400" />
            </div>
            <div class="flex-1">
              <div class="flex items-center gap-3 mb-2">
                <h1 class="text-2xl lg:text-3xl font-bold text-gray-900 dark:text-white">{{ v.description?.moniker }}</h1>
                <span v-if="v.status === 'BOND_STATUS_BONDED'" class="px-3 py-1 rounded-full text-xs font-semibold bg-emerald-500/10 text-emerald-600 dark:text-emerald-400">Active</span>
                <span v-else-if="v.jailed" class="px-3 py-1 rounded-full text-xs font-semibold bg-red-500/10 text-red-600 dark:text-red-400">Jailed</span>
                <span v-else class="px-3 py-1 rounded-full text-xs font-semibold bg-gray-500/10 text-gray-600 dark:text-gray-400">Inactive</span>
              </div>
              <p class="text-gray-600 dark:text-gray-400 mb-3 line-clamp-2">{{ v.description?.details || 'No description available' }}</p>
              <div class="flex flex-wrap gap-3">
                <a v-if="v.description?.website" :href="v.description.website" target="_blank" class="flex items-center gap-1.5 text-sm text-primary hover:underline">
                  <Icon icon="mdi:web" />
                  {{ v.description.website }}
                </a>
                <span v-if="v.description?.security_contact" class="flex items-center gap-1.5 text-sm text-gray-500 dark:text-gray-400">
                  <Icon icon="mdi:email" />
                  {{ v.description.security_contact }}
                </span>
              </div>
            </div>
          </div>

          <!-- Right: Action Button -->
          <div class="flex items-center lg:ml-auto">
            <button
              class="px-6 py-3 rounded-xl bg-gradient-to-r from-primary to-emerald-600 text-white font-semibold hover:from-primary/90 hover:to-emerald-600/90 transition-all shadow-lg shadow-primary/20 flex items-center gap-2"
              @click="dialog.open('delegate', { validator_address: v.operator_address })"
            >
              <Icon icon="mdi:hand-coin" class="text-xl" />
              Delegate
            </button>
          </div>
        </div>
        
        <!-- Live Blocks Section - Responsive Grid -->
        <div class="mt-6 pt-6 border-t border-gray-200/50 dark:border-gray-700/50">
          <div class="flex items-center justify-between mb-3">
            <div class="flex items-center gap-2">
              <Icon icon="mdi:cube-outline" class="text-lg text-primary" />
              <span class="text-sm font-semibold text-gray-700 dark:text-gray-300">Block Signing History</span>
            </div>
            <div class="flex items-center gap-2">
              <div class="w-2 h-2 rounded-full bg-emerald-500 animate-pulse"></div>
              <span class="text-xs text-gray-500 dark:text-gray-400">Live</span>
              <span class="text-xs text-gray-400 dark:text-gray-500 ml-2">{{ liveBlocks.length }}/100</span>
            </div>
          </div>
          
          <div v-if="loadingBlocks" class="grid grid-cols-20 sm:grid-cols-25 md:grid-cols-50 gap-[2px]">
            <div
              v-for="i in 100"
              :key="i"
              class="aspect-square rounded-[2px] bg-gray-200 dark:bg-gray-700 animate-pulse"
            ></div>
          </div>
          <div v-else class="grid grid-cols-20 sm:grid-cols-25 md:grid-cols-50 gap-[2px]">
            <div
              v-for="(block, idx) in liveBlocks"
              :key="block.height"
              class="aspect-square rounded-[2px] transition-all duration-200 cursor-pointer hover:scale-150 hover:z-10"
              :class="block.signed ? 'bg-emerald-500 hover:bg-emerald-400' : 'bg-red-500 hover:bg-red-400'"
              :title="'#' + block.height + (block.signed ? ' Signed' : ' Missed')"
            ></div>
            <div
              v-for="i in Math.max(0, 100 - liveBlocks.length)"
              :key="'empty-' + i"
              class="aspect-square rounded-[2px] bg-gray-200 dark:bg-gray-700"
            ></div>
          </div>
          
          <div class="flex items-center gap-4 mt-3 text-xs text-gray-500 dark:text-gray-400">
            <span class="flex items-center gap-1.5"><span class="w-2.5 h-2.5 rounded-[2px] bg-emerald-500"></span> Signed</span>
            <span class="flex items-center gap-1.5"><span class="w-2.5 h-2.5 rounded-[2px] bg-red-500"></span> Missed</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Stats Grid -->
    <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-6 gap-4">
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4 text-center">
        <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ format.formatToken2({ amount: v.tokens, denom: staking.params.bond_denom }) }}</div>
        <div class="text-xs text-gray-500 dark:text-gray-400 mt-1">Total Staked</div>
      </div>
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4 text-center">
        <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ commissionRate.toFixed(1) }}%</div>
        <div class="text-xs text-gray-500 dark:text-gray-400 mt-1">Commission</div>
      </div>
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4 text-center">
        <div class="text-2xl font-bold" :class="getUptimeColor(uptime)">{{ uptime !== null ? uptime.toFixed(2) + '%' : 'N/A' }}</div>
        <div class="text-xs text-gray-500 dark:text-gray-400 mt-1">Uptime</div>
      </div>
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4 text-center">
        <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ apr }}</div>
        <div class="text-xs text-gray-500 dark:text-gray-400 mt-1">APR</div>
      </div>
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4 text-center">
        <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ selfRate }}</div>
        <div class="text-xs text-gray-500 dark:text-gray-400 mt-1">Self Bonded</div>
      </div>
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/60 border border-gray-200/50 dark:border-gray-700/50 p-4 text-center">
        <div class="text-2xl font-bold text-gray-900 dark:text-white">{{ v.unbonding_height || '0' }}</div>
        <div class="text-xs text-gray-500 dark:text-gray-400 mt-1">Unbonding Height</div>
      </div>
    </div>

    <!-- Commission Rate & Rewards Section -->
    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
      <!-- Commission Rate Card -->
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200/50 dark:border-gray-700/50 p-6">
        <div class="flex items-center gap-3 mb-6">
          <div class="p-2.5 rounded-xl bg-gradient-to-br from-amber-500 to-orange-500">
            <Icon icon="mdi:percent" class="text-xl text-white" />
          </div>
          <div>
            <h3 class="font-semibold text-gray-900 dark:text-white">Commission Rate</h3>
            <p class="text-xs text-gray-500 dark:text-gray-400">Updated {{ format.toDay(v.commission?.update_time, 'from') }}</p>
          </div>
        </div>
        
        <!-- Visual Commission Display -->
        <div class="space-y-4">
          <!-- Current Rate -->
          <div>
            <div class="flex justify-between mb-2">
              <span class="text-sm text-gray-600 dark:text-gray-400">Current Rate</span>
              <span class="text-sm font-semibold text-gray-900 dark:text-white">{{ commissionRate.toFixed(1) }}%</span>
            </div>
            <div class="h-3 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
              <div class="h-full bg-gradient-to-r from-emerald-500 to-green-500 rounded-full transition-all" :style="{ width: commissionRate + '%' }"></div>
            </div>
          </div>
          
          <!-- Max Rate -->
          <div>
            <div class="flex justify-between mb-2">
              <span class="text-sm text-gray-600 dark:text-gray-400">Max Rate</span>
              <span class="text-sm font-semibold text-gray-900 dark:text-white">{{ maxRate.toFixed(1) }}%</span>
            </div>
            <div class="h-3 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
              <div class="h-full bg-gradient-to-r from-blue-500 to-cyan-500 rounded-full transition-all" :style="{ width: maxRate + '%' }"></div>
            </div>
          </div>
          
          <!-- Max Change Rate -->
          <div>
            <div class="flex justify-between mb-2">
              <span class="text-sm text-gray-600 dark:text-gray-400">Max Daily Change</span>
              <span class="text-sm font-semibold text-gray-900 dark:text-white">{{ maxChangeRate.toFixed(1) }}%</span>
            </div>
            <div class="h-3 bg-gray-200 dark:bg-gray-700 rounded-full overflow-hidden">
              <div class="h-full bg-gradient-to-r from-purple-500 to-pink-500 rounded-full transition-all" :style="{ width: maxChangeRate + '%' }"></div>
            </div>
          </div>
        </div>
      </div>

      <!-- Commissions & Rewards Card -->
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200/50 dark:border-gray-700/50 p-6">
        <div class="flex items-center gap-3 mb-6">
          <div class="p-2.5 rounded-xl bg-gradient-to-br from-emerald-500 to-green-600">
            <Icon icon="mdi:cash-multiple" class="text-xl text-white" />
          </div>
          <h3 class="font-semibold text-gray-900 dark:text-white">Commissions & Rewards</h3>
        </div>
        
        <div class="space-y-4 max-h-48 overflow-auto">
          <div>
            <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mb-2">Commissions</div>
            <div class="flex flex-wrap gap-2">
              <span v-for="(c, k) in commission" :key="'c-'+k" class="px-3 py-1.5 rounded-lg text-xs font-medium bg-emerald-500/10 text-emerald-600 dark:text-emerald-400">
                {{ format.formatToken2(c) }}
              </span>
              <span v-if="!commission?.length" class="text-sm text-gray-400">No commissions</span>
            </div>
          </div>
          <div>
            <div class="text-xs text-gray-500 dark:text-gray-400 uppercase tracking-wide mb-2">Outstanding Rewards</div>
            <div class="flex flex-wrap gap-2">
              <span v-for="(r, k) in rewards" :key="'r-'+k" class="px-3 py-1.5 rounded-lg text-xs font-medium bg-blue-500/10 text-blue-600 dark:text-blue-400">
                {{ format.formatToken2(r) }}
              </span>
              <span v-if="!rewards?.length" class="text-sm text-gray-400">No rewards</span>
            </div>
          </div>
        </div>
        
        <button
          class="w-full mt-4 py-2.5 rounded-xl bg-gradient-to-r from-emerald-500 to-green-600 text-white font-semibold text-sm hover:from-emerald-600 hover:to-green-700 transition-all"
          @click="dialog.open('withdraw_commission', { validator_address: v.operator_address })"
        >
          <Icon icon="mdi:bank-transfer-out" class="inline mr-2" />
          Withdraw Commission
        </button>
      </div>

      <!-- Addresses Card -->
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200/50 dark:border-gray-700/50 p-6">
        <div class="flex items-center gap-3 mb-6">
          <div class="p-2.5 rounded-xl bg-gradient-to-br from-blue-500 to-cyan-600">
            <Icon icon="mdi:card-account-details" class="text-xl text-white" />
          </div>
          <h3 class="font-semibold text-gray-900 dark:text-white">Addresses</h3>
        </div>
        
        <div class="space-y-3 max-h-56 overflow-auto">
          <div class="group">
            <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Account Address</div>
            <div class="flex items-center gap-2">
              <RouterLink :to="`/${chain}/account/${addresses.account}`" class="text-xs text-primary hover:underline truncate flex-1">
                {{ addresses.account }}
              </RouterLink>
              <button @click="copyToClipboard(addresses.account)" class="p-1.5 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700 opacity-0 group-hover:opacity-100 transition-opacity">
                <Icon icon="mdi:content-copy" class="text-gray-400" />
              </button>
            </div>
          </div>
          <div class="group">
            <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Operator Address</div>
            <div class="flex items-center gap-2">
              <span class="text-xs text-gray-700 dark:text-gray-300 truncate flex-1">{{ v.operator_address }}</span>
              <button @click="copyToClipboard(v.operator_address)" class="p-1.5 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700 opacity-0 group-hover:opacity-100 transition-opacity">
                <Icon icon="mdi:content-copy" class="text-gray-400" />
              </button>
            </div>
          </div>
          <div class="group">
            <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Hex Address</div>
            <div class="flex items-center gap-2">
              <span class="text-xs text-gray-700 dark:text-gray-300 truncate flex-1">{{ addresses.hex }}</span>
              <button @click="copyToClipboard(addresses.hex)" class="p-1.5 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700 opacity-0 group-hover:opacity-100 transition-opacity">
                <Icon icon="mdi:content-copy" class="text-gray-400" />
              </button>
            </div>
          </div>
          <div class="group">
            <div class="text-xs text-gray-500 dark:text-gray-400 mb-1">Consensus Address</div>
            <div class="flex items-center gap-2">
              <span class="text-xs text-gray-700 dark:text-gray-300 truncate flex-1">{{ addresses.valCons }}</span>
              <button @click="copyToClipboard(addresses.valCons)" class="p-1.5 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700 opacity-0 group-hover:opacity-100 transition-opacity">
                <Icon icon="mdi:content-copy" class="text-gray-400" />
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Transactions & Voting Power Events Side by Side -->
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
      <!-- Transactions -->
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center justify-between px-6 py-4 border-b border-gray-100 dark:border-gray-700/50">
          <div class="flex items-center gap-3">
            <div class="p-2.5 rounded-xl bg-gradient-to-br from-purple-500 to-pink-500">
              <Icon icon="mdi:swap-horizontal" class="text-xl text-white" />
            </div>
            <div>
              <h3 class="font-semibold text-gray-900 dark:text-white">Transactions</h3>
              <p class="text-xs text-gray-500 dark:text-gray-400">{{ txs.tx_responses?.length || 0 }} total</p>
            </div>
          </div>
        </div>
        
        <div class="divide-y divide-gray-100 dark:divide-gray-700/50">
          <div v-for="(item, i) in displayedTxs" :key="i" class="px-6 py-4 hover:bg-gray-50 dark:hover:bg-gray-700/30 transition-colors">
            <div class="flex items-start justify-between gap-4">
              <div class="flex-1 min-w-0">
                <div class="flex items-center gap-2 mb-1">
                  <RouterLink :to="`/${props.chain}/tx/${item.txhash}`" class="text-sm font-medium text-primary hover:underline truncate">
                    {{ item.txhash?.slice(0, 16) }}...
                  </RouterLink>
                  <Icon v-if="item.code === 0" icon="mdi:check-circle" class="text-emerald-500" />
                  <Icon v-else icon="mdi:close-circle" class="text-red-500" />
                </div>
                <div class="text-xs text-gray-500 dark:text-gray-400">{{ format.messages(item.tx.body.messages) }}</div>
              </div>
              <div class="text-right flex-shrink-0">
                <RouterLink :to="`/${props.chain}/block/${item.height}`" class="text-xs text-primary hover:underline">
                  #{{ item.height }}
                </RouterLink>
                <div class="text-xs text-gray-400">{{ format.toDay(item.timestamp, 'from') }}</div>
              </div>
            </div>
          </div>
          <div v-if="!txs.tx_responses?.length" class="px-6 py-8 text-center text-gray-400">
            <Icon icon="mdi:swap-horizontal" class="text-4xl mb-2 opacity-50" />
            <p>No transactions found</p>
          </div>
        </div>
        
        <div v-if="txs.tx_responses?.length > 5" class="px-6 py-4 border-t border-gray-100 dark:border-gray-700/50">
          <button 
            @click="showAllTxs = !showAllTxs"
            class="w-full py-2 rounded-xl bg-gray-100 dark:bg-gray-700/50 text-gray-600 dark:text-gray-300 font-medium text-sm hover:bg-gray-200 dark:hover:bg-gray-600/50 transition-colors"
          >
            {{ showAllTxs ? 'Show Less' : `View More (${txs.tx_responses.length - 5} more)` }}
          </button>
        </div>
      </div>

      <!-- Voting Power Events -->
      <div class="relative overflow-hidden rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200/50 dark:border-gray-700/50">
        <div class="flex items-center justify-between px-6 py-4 border-b border-gray-100 dark:border-gray-700/50">
          <div class="flex items-center gap-3">
            <div class="p-2.5 rounded-xl bg-gradient-to-br from-blue-500 to-indigo-600">
              <Icon icon="mdi:chart-timeline-variant" class="text-xl text-white" />
            </div>
            <div>
              <h3 class="font-semibold text-gray-900 dark:text-white">Voting Power Events</h3>
              <p class="text-xs text-gray-500 dark:text-gray-400">{{ events.pagination?.total || 0 }} total</p>
            </div>
          </div>
          <div class="flex gap-1">
            <button 
              @click="loadPowerEvents(1, EventType.Delegate)"
              class="px-3 py-1.5 rounded-lg text-xs font-medium transition-colors"
              :class="selectedEventType === EventType.Delegate ? 'bg-emerald-500/10 text-emerald-600 dark:text-emerald-400' : 'text-gray-500 hover:bg-gray-100 dark:hover:bg-gray-700'"
            >
              Delegate
            </button>
            <button 
              @click="loadPowerEvents(1, EventType.Unbond)"
              class="px-3 py-1.5 rounded-lg text-xs font-medium transition-colors"
              :class="selectedEventType === EventType.Unbond ? 'bg-red-500/10 text-red-600 dark:text-red-400' : 'text-gray-500 hover:bg-gray-100 dark:hover:bg-gray-700'"
            >
              Unbond
            </button>
          </div>
        </div>
        
        <div class="divide-y divide-gray-100 dark:divide-gray-700/50">
          <div v-for="(item, i) in displayedEvents" :key="i" class="px-6 py-4 hover:bg-gray-50 dark:hover:bg-gray-700/30 transition-colors">
            <div class="flex items-start justify-between gap-4">
              <div class="flex-1 min-w-0">
                <div class="flex items-center gap-2 mb-1">
                  <RouterLink v-for="d in mapDelegators(item.tx?.body?.messages)" :key="d" :to="`/${props.chain}/account/${d}`" class="text-sm font-medium text-primary hover:underline truncate">
                    {{ d?.slice(0, 20) }}...
                  </RouterLink>
                </div>
                <div class="flex items-center gap-2">
                  <span :class="selectedEventType === EventType.Delegate ? 'text-emerald-500' : 'text-red-500'" class="text-sm font-semibold">
                    {{ selectedEventType === EventType.Delegate ? '+' : '-' }}{{ mapEvents(item.events) }}
                  </span>
                  <Icon v-if="item.code === 0" icon="mdi:check-circle" class="text-emerald-500 text-sm" />
                  <Icon v-else icon="mdi:close-circle" class="text-red-500 text-sm" />
                </div>
              </div>
              <div class="text-right flex-shrink-0">
                <RouterLink :to="`/${props.chain}/block/${item.height}`" class="text-xs text-primary hover:underline">
                  #{{ item.height }}
                </RouterLink>
                <div class="text-xs text-gray-400">{{ format.toDay(item.timestamp, 'from') }}</div>
              </div>
            </div>
          </div>
          <div v-if="!events.tx_responses?.length" class="px-6 py-8 text-center text-gray-400">
            <Icon icon="mdi:chart-timeline-variant" class="text-4xl mb-2 opacity-50" />
            <p>No events found</p>
          </div>
        </div>
        
        <div v-if="events.tx_responses?.length > 5" class="px-6 py-4 border-t border-gray-100 dark:border-gray-700/50">
          <button 
            @click="showAllEvents = !showAllEvents"
            class="w-full py-2 rounded-xl bg-gray-100 dark:bg-gray-700/50 text-gray-600 dark:text-gray-300 font-medium text-sm hover:bg-gray-200 dark:hover:bg-gray-600/50 transition-colors"
          >
            {{ showAllEvents ? 'Show Less' : `View More` }}
          </button>
        </div>
      </div>
    </div>

    <!-- Toast Notifications -->
    <div v-if="showCopyToast === 1" class="fixed bottom-4 right-4 px-4 py-3 rounded-xl bg-emerald-500 text-white shadow-lg flex items-center gap-2 animate-fade-in">
      <Icon icon="mdi:check-circle" />
      <span>Copied to clipboard!</span>
    </div>
    <div v-if="showCopyToast === 2" class="fixed bottom-4 right-4 px-4 py-3 rounded-xl bg-red-500 text-white shadow-lg flex items-center gap-2 animate-fade-in">
      <Icon icon="mdi:close-circle" />
      <span>Failed to copy</span>
    </div>
  </div>
</template>

<style scoped>
@keyframes fade-in {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}
.animate-fade-in {
  animation: fade-in 0.3s ease-out;
}
</style>
