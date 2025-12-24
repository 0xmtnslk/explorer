<script lang="ts" setup>
import { ref, onMounted, computed, onUnmounted } from 'vue';
import { fromHex, toBase64, fromBase64, toHex } from '@cosmjs/encoding';
import { useStakingStore, useBaseStore, useBlockchain, useFormatter } from '@/stores';
import UptimeBar from '@/components/UptimeBar.vue';
import type { SlashingParam, SigningInfo, Block } from '@/types';
import { consensusPubkeyToHexAddress, valconsToBase64 } from '@/libs';
import { Icon } from '@iconify/vue';

const props = defineProps(['chain']);

const stakingStore = useStakingStore();
const format = useFormatter();
const baseStore = useBaseStore();
const chainStore = useBlockchain();
const latest = ref(0);
const keyword = ref('');
const live = ref(true);
const slashingParam = ref({} as SlashingParam);
const signingInfo = ref({} as Record<string, SigningInfo>);
const consumerValidators = ref([] as { moniker: string; base64: string }[]);

interface BlockColor {
  height: string;
  color: string;
}
interface ValidatorUnit {
  moniker: string;
  blocks: BlockColor[];
  hex: string;
  base64: string;
  missed_blocks_counter: number | string;
  uptime: number;
  signing: SigningInfo;
}

function padding(blocks: BlockColor[] = []) {
  const raw = Array(50)
    .fill({ height: '0', color: 'bg-secondary' } as BlockColor)
    .concat(blocks);
  return raw.slice(raw.length - 50);
}

const validatorSet = computed(() => {
  if (chainStore.isConsumerChain) {
    return consumerValidators.value.map((v) => {
      const b64 = valconsToBase64(v.moniker);
      const moniker = stakingStore.validators.find(
        (x) => toBase64(fromHex(consensusPubkeyToHexAddress(x.consensus_pubkey))) === b64
      )?.description.moniker;
      return {
        moniker: moniker || v.moniker,
        base64: v.base64,
      };
    });
  }
  return stakingStore.validators.map((v) => {
    const hex = consensusPubkeyToHexAddress(v.consensus_pubkey);
    return {
      moniker: v.description.moniker,
      base64: toBase64(fromHex(hex)),
    };
  });
});

const blockColors = ref({} as Record<string, BlockColor[]>);

const grid = computed(() => {
  const validators =
    keyword.value.length === 0
      ? validatorSet.value
      : validatorSet.value.filter((v) => v.moniker.toLowerCase().includes(keyword.value.toLowerCase()));

  const window = Number(slashingParam.value.signed_blocks_window || 0);
  return validators.map((v) => {
    const signing = signingInfo.value[v.base64];
    const uptime = signing && window > 0 ? (window - Number(signing.missed_blocks_counter)) / window : undefined;
    return {
      moniker: v.moniker,
      base64: v.base64,
      blocks: padding(blockColors.value[v.base64] || []),
      uptime,
      missed_blocks_counter: signing?.missed_blocks_counter,
      signing,
    } as ValidatorUnit;
  });
});

const preload = ref(false);
baseStore.$subscribe((_, state) => {
  const newHeight = Number(state.latest?.block?.header?.height || 0);
  if (newHeight > latest.value) {
    latest.value = newHeight;
    if (!preload.value) {
      preFill();
      preload.value = true;
    }

    if (newHeight > 0 && consumerValidators.value.length === 0) {
      const chain_id = state.latest.block.header.chain_id;
      Promise.resolve().then(async () => {
        await stakingStore.getConsumerValidators(chain_id).then((x) => {
          x.validators
            .sort((a, b) => Number(b.power) - Number(a.power))
            .forEach((v) => {
              const base64 = toBase64(
                fromHex(
                  consensusPubkeyToHexAddress({ '@type': '/cosmos.crypto.ed25519.PubKey', key: v.consumer_key.ed25519 })
                )
              );
              const moniker = v.provider_address;
              consumerValidators.value.push({ moniker, base64 });
            });
        });
      });
    }

    if (Number(state.latest.block.header.height) % 7 === 0) updateTotalSigningInfo();
    fillblock(state.latest);
  }
});

onMounted(() => {
  live.value = true;
  baseStore.recents?.forEach((b) => {
    fillblock(b, 'start');
  });
  updateTotalSigningInfo();
  chainStore.rpc.getSlashingParams().then((x) => {
    slashingParam.value = x.params;
  });
});

function preFill() {
  if (latest.value > 50 && baseStore.recents.length >= 49) return;
  let promise = Promise.resolve();
  for (let i = latest.value - baseStore.recents.length; i > latest.value - 50 && i > 1; i -= 1) {
    promise = promise.then(
      () =>
        new Promise((resolve) => {
          if (live.value) {
            if (i > latest.value - 50)
              baseStore.fetchBlock(i).then((x) => {
                fillblock(x, 'start');
                resolve();
              });
          }
        })
    );
  }
}

function fillblock(b: Block, direction: string = 'end') {
  validatorSet.value.forEach((v) => {
    const sig = b.block.last_commit?.signatures.find((s) => s.validator_address === v.base64);
    const block = blockColors.value[v.base64] || [];
    let color = {
      height: b.block.header.height,
      color: 'bg-red-500',
    };
    if (sig) {
      color = {
        height: b.block.header.height,
        color: sig.block_id_flag === 'BLOCK_ID_FLAG_COMMIT' ? 'bg-green-500' : 'bg-yellow-500',
      };
    }
    if (direction === 'end') {
      block.push(color);
    } else {
      block.unshift(color);
    }
    if (block.length > 50) block.shift();
    blockColors.value[v.base64] = block;
  });
}

function updateTotalSigningInfo() {
  chainStore.rpc.getSlashingSigningInfos().then((x) => {
    x.info?.forEach((i) => {
      signingInfo.value[valconsToBase64(i.address)] = i;
    });
  });
}

onUnmounted(() => {
  live.value = false;
});

const tab = ref('blocks');
</script>

<template>
  <div class="space-y-6">
    <!-- Hero Section -->
    <div class="relative overflow-hidden rounded-2xl bg-gradient-to-r from-cyan-500 via-blue-500 to-purple-600 p-6 md:p-8">
      <div class="absolute inset-0 bg-black/10"></div>
      <div class="absolute -top-24 -right-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      <div class="absolute -bottom-24 -left-24 w-64 h-64 bg-white/10 rounded-full blur-3xl"></div>
      
      <div class="relative z-10">
        <div class="flex items-center gap-3 mb-2">
          <div class="w-12 h-12 rounded-xl bg-white/20 backdrop-blur flex items-center justify-center">
            <Icon icon="mdi:chart-timeline-variant" class="text-2xl text-white" />
          </div>
          <div>
            <h1 class="text-2xl md:text-3xl font-bold text-white">Validator Uptime</h1>
            <p class="text-white/80 text-sm">Monitor validator performance and block signing</p>
          </div>
        </div>
        
        <!-- Stats -->
        <div class="flex flex-wrap gap-4 mt-4">
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Latest Block</div>
            <div class="text-white font-bold text-lg">#{{ latest.toLocaleString() }}</div>
          </div>
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Validators</div>
            <div class="text-white font-bold text-lg">{{ validatorSet.length }}</div>
          </div>
          <div class="bg-white/10 backdrop-blur rounded-xl px-4 py-2">
            <div class="text-white/70 text-xs">Min Uptime</div>
            <div class="text-white font-bold text-lg">{{ format.percent(slashingParam.min_signed_per_window) || '-' }}</div>
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
            ? 'bg-gradient-to-r from-cyan-500 to-blue-500 text-white shadow-md' 
            : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
        >
          <Icon icon="mdi:view-grid" class="text-lg" />
          {{ $t('module.blocks') }}
        </button>
        <button
          @click="tab = 'overall'"
          class="flex items-center gap-2 px-4 py-2.5 rounded-xl text-sm font-medium transition-all"
          :class="tab === 'overall' 
            ? 'bg-gradient-to-r from-cyan-500 to-blue-500 text-white shadow-md' 
            : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'"
        >
          <Icon icon="mdi:chart-bar" class="text-lg" />
          {{ $t('uptime.overall') }}
        </button>
        <RouterLink :to="`/${chain}/uptime/customize`" class="flex items-center gap-2 px-4 py-2.5 rounded-xl text-sm font-medium text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700 transition-all">
          <Icon icon="mdi:cog" class="text-lg" />
          {{ $t('uptime.customize') }}
        </RouterLink>
      </div>
    </div>

    <!-- Search -->
    <div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-4">
      <div class="relative">
        <Icon icon="mdi:magnify" class="absolute left-4 top-1/2 -translate-y-1/2 text-gray-400" />
        <input
          type="text"
          v-model="keyword"
          placeholder="Search validators by name..."
          class="w-full pl-12 pr-4 py-3 bg-gray-100 dark:bg-gray-700 border-0 rounded-xl focus:ring-2 focus:ring-cyan-500/50 text-gray-700 dark:text-gray-300 placeholder-gray-400 transition-all"
        />
      </div>
    </div>

    <!-- Blocks View -->
    <div v-if="tab === 'blocks'" class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6">
      <!-- Legend -->
      <div class="flex flex-wrap items-center gap-4 mb-6 pb-4 border-b border-gray-200 dark:border-gray-700">
        <span class="text-sm font-medium text-gray-600 dark:text-gray-400">{{ $t('uptime.legend') }}:</span>
        <div class="flex items-center gap-2">
          <span class="w-4 h-4 rounded bg-green-500"></span>
          <span class="text-sm text-gray-600 dark:text-gray-400">{{ $t('uptime.committed') }}</span>
        </div>
        <div class="flex items-center gap-2">
          <span class="w-4 h-4 rounded bg-yellow-500"></span>
          <span class="text-sm text-gray-600 dark:text-gray-400">{{ $t('uptime.precommitted') }}</span>
        </div>
        <div class="flex items-center gap-2">
          <span class="w-4 h-4 rounded bg-red-500"></span>
          <span class="text-sm text-gray-600 dark:text-gray-400">{{ $t('uptime.missed') }}</span>
        </div>
      </div>

      <!-- Validator Grid -->
      <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 2xl:grid-cols-4 gap-4">
        <div 
          v-for="(unit, i) in grid" 
          :key="i"
          class="bg-gray-50 dark:bg-gray-700/50 rounded-xl p-4 hover:shadow-md transition-all border border-gray-100 dark:border-gray-600"
        >
          <div class="flex items-center justify-between mb-3">
            <div class="flex items-center gap-2 min-w-0 flex-1">
              <span class="flex-shrink-0 w-6 h-6 rounded-full bg-gradient-to-r from-cyan-500 to-blue-500 text-white text-xs flex items-center justify-center font-medium">
                {{ i + 1 }}
              </span>
              <span class="text-sm font-medium text-gray-800 dark:text-white truncate">{{ unit.moniker }}</span>
            </div>
            <span 
              class="flex-shrink-0 px-2 py-0.5 rounded-full text-xs font-bold"
              :class="Number(unit?.missed_blocks_counter || 0) > 10 
                ? 'bg-red-100 dark:bg-red-900/30 text-red-600 dark:text-red-400' 
                : 'bg-green-100 dark:bg-green-900/30 text-green-600 dark:text-green-400'"
            >
              {{ unit?.missed_blocks_counter || 0 }} missed
            </span>
          </div>
          <UptimeBar :blocks="unit.blocks" />
        </div>
      </div>

      <div v-if="grid.length === 0" class="text-center py-12">
        <Icon icon="mdi:server-off" class="text-5xl text-gray-300 dark:text-gray-600 mb-3" />
        <p class="text-gray-500">No validators found</p>
      </div>
    </div>

    <!-- Overall View -->
    <div v-if="tab === 'overall'" class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 overflow-hidden">
      <div class="overflow-x-auto">
        <table class="w-full">
          <thead>
            <tr class="bg-gradient-to-r from-gray-50 to-gray-100 dark:from-gray-700 dark:to-gray-800">
              <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('account.validator') }}</th>
              <th class="text-right px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('module.uptime') }}</th>
              <th class="text-left px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.last_jailed_time') }}</th>
              <th class="text-right px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.signed_precommits') }}</th>
              <th class="text-right px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.start_height') }}</th>
              <th class="text-center px-6 py-4 text-xs font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">{{ $t('uptime.tombstoned') }}</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-100 dark:divide-gray-700">
            <tr v-for="(v, i) in grid" :key="i" class="hover:bg-gray-50 dark:hover:bg-gray-700/50 transition-colors">
              <td class="px-6 py-4">
                <div class="flex items-center gap-3">
                  <span class="w-8 h-8 rounded-full bg-gradient-to-r from-cyan-500 to-blue-500 text-white text-sm flex items-center justify-center font-medium">
                    {{ i + 1 }}
                  </span>
                  <span class="text-sm font-medium text-gray-800 dark:text-white max-w-[200px] truncate">{{ v.moniker }}</span>
                </div>
              </td>
              <td class="px-6 py-4 text-right">
                <div class="flex items-center justify-end gap-2">
                  <div class="w-20 h-2 bg-gray-200 dark:bg-gray-600 rounded-full overflow-hidden">
                    <div 
                      class="h-full rounded-full transition-all"
                      :class="v.uptime && v.uptime > 0.95 ? 'bg-green-500' : 'bg-red-500'"
                      :style="{ width: `${(v.uptime || 0) * 100}%` }"
                    ></div>
                  </div>
                  <span 
                    class="text-sm font-medium min-w-[60px]"
                    :class="v.uptime && v.uptime > 0.95 ? 'text-green-600 dark:text-green-400' : 'text-red-600 dark:text-red-400'"
                  >
                    {{ format.percent(v.uptime) }}
                  </span>
                </div>
              </td>
              <td class="px-6 py-4">
                <span v-if="v.signing && !v.signing.jailed_until.startsWith('1970')" class="text-sm text-gray-600 dark:text-gray-400">
                  {{ format.toDay(v.signing.jailed_until, 'from') }}
                </span>
                <span v-else class="text-sm text-gray-400">-</span>
              </td>
              <td class="px-6 py-4 text-right">
                <span class="text-sm text-gray-600 dark:text-gray-400">{{ v.signing?.index_offset || '-' }}</span>
              </td>
              <td class="px-6 py-4 text-right">
                <span class="text-sm text-gray-600 dark:text-gray-400">{{ v.signing?.start_height || '-' }}</span>
              </td>
              <td class="px-6 py-4 text-center">
                <span 
                  v-if="v.signing?.tombstoned === true"
                  class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-red-100 dark:bg-red-900/30 text-red-700 dark:text-red-400"
                >
                  <Icon icon="mdi:skull" class="mr-1" />
                  Yes
                </span>
                <span 
                  v-else
                  class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 dark:bg-green-900/30 text-green-700 dark:text-green-400"
                >
                  <Icon icon="mdi:check-circle" class="mr-1" />
                  No
                </span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Footer Info -->
      <div class="px-6 py-4 bg-gray-50 dark:bg-gray-700/50 border-t border-gray-200 dark:border-gray-700">
        <div class="flex flex-wrap items-center justify-between gap-4">
          <div class="flex items-center gap-2 text-sm text-gray-600 dark:text-gray-400">
            <Icon icon="mdi:information-outline" class="text-lg" />
            Window Size: <span class="font-medium text-gray-800 dark:text-white">{{ slashingParam.signed_blocks_window || '-' }}</span>
          </div>
          <div class="flex items-center gap-2">
            <span class="text-sm text-gray-600 dark:text-gray-400">{{ $t('uptime.minimum_uptime') }}:</span>
            <span class="px-3 py-1 bg-red-100 dark:bg-red-900/30 text-red-600 dark:text-red-400 rounded-lg text-sm font-bold">
              {{ format.percent(slashingParam.min_signed_per_window) }}
            </span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<route>
  {
    meta: {
      i18n: 'uptime',
      order: 8
    }
  }
</route>
