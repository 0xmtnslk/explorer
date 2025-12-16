<script lang="ts" setup>
import { computed, ref, reactive, onMounted } from 'vue';
import MdEditor from 'md-editor-v3';
import ObjectElement from '@/components/dynamic/ObjectElement.vue';
import { useBaseStore, useBlockchain, useFormatter, useGovStore, useStakingStore, useTxDialog } from '@/stores';
import { PageRequest, type GovProposal, type GovVote, type PaginatedProposalDeposit, type Pagination } from '@/types';
import PaginationBar from '@/components/PaginationBar.vue';
import { fromBech32, toHex } from '@cosmjs/encoding';
import { Icon } from '@iconify/vue';

const props = defineProps(['proposal_id', 'chain']);
const proposal = ref({} as GovProposal);
const format = useFormatter();
const store = useGovStore();
const dialog = useTxDialog();
const stakingStore = useStakingStore();
const chainStore = useBlockchain();

const votesTab = ref<'validators' | 'users'>('validators');

store.fetchProposal(props.proposal_id).then((res) => {
  let proposalDetail = reactive(res.proposal);
  if (res.proposal?.status === 'PROPOSAL_STATUS_VOTING_PERIOD') {
    store.fetchTally(props.proposal_id).then((tallRes) => {
      proposalDetail.final_tally_result = tallRes?.tally;
    });
  }
  proposal.value = proposalDetail;
  if (proposalDetail.content?.changes) {
    proposalDetail.content?.changes.forEach((item) => {
      chainStore.rpc.getParams(item.subspace, item.key).then((res) => {
        if (proposal.value.content && res.param) {
          if (proposal.value.content.current) {
            proposal.value.content.current.push(res.param);
          } else {
            proposal.value.content.current = [res.param];
          }
        }
      });
    });
  }

  const msgType = proposalDetail.content?.['@type'] || '';
  if (msgType.endsWith('MsgUpdateParams')) {
    if (msgType.indexOf('staking') > -1) {
      chainStore.rpc.getStakingParams().then((res) => {
        addCurrentParams(res);
      });
    } else if (msgType.indexOf('gov') > -1) {
      chainStore.rpc.getGovParamsVoting().then((res) => {
        addCurrentParams(res);
      });
    } else if (msgType.indexOf('distribution') > -1) {
      chainStore.rpc.getDistributionParams().then((res) => {
        addCurrentParams(res);
      });
    } else if (msgType.indexOf('slashing') > -1) {
      chainStore.rpc.getSlashingParams().then((res) => {
        addCurrentParams(res);
      });
    }
  }
});

function addCurrentParams(res: any) {
  if (proposal.value.content && res.params) {
    proposal.value.content.params = [proposal.value.content?.params];
    proposal.value.content.current = [res.params];
  }
}

const color = computed(() => {
  if (proposal.value.status === 'PROPOSAL_STATUS_PASSED') {
    return 'success';
  } else if (proposal.value.status === 'PROPOSAL_STATUS_REJECTED') {
    return 'error';
  }
  return 'voting';
});

const status = computed(() => {
  if (proposal.value.status) {
    return proposal.value.status.replace('PROPOSAL_STATUS_', '');
  }
  return '';
});

const proposalTitle = computed(() => {
  return proposal.value.title || 
         proposal.value.content?.title || 
         proposal.value.content?.plan?.name ||
         metaItem(proposal.value?.metadata)?.title ||
         `Proposal #${props.proposal_id}`;
});

const deposit = ref({} as PaginatedProposalDeposit);
store.fetchProposalDeposits(props.proposal_id).then((x) => (deposit.value = x));

const votes = ref({} as GovVote[]);
const pageRequest = ref(new PageRequest());
const pageResponse = ref({} as Pagination);

store.fetchProposalVotes(props.proposal_id).then((x) => {
  votes.value = x.votes;
  pageResponse.value = x.pagination;
});

onMounted(() => {
  stakingStore.init();
});

function shortTime(v: string) {
  if (v) {
    return format.toDay(v, 'from');
  }
  return '';
}

const votingCountdown = computed((): number => {
  const now = new Date();
  const end = new Date(proposal.value.voting_end_time);
  return end.getTime() - now.getTime();
});

const upgradeCountdown = computed((): number => {
  const height = Number(proposal.value.content?.plan?.height || 0);
  if (height > 0) {
    const base = useBaseStore();
    const current = Number(base.latest?.block?.header?.height || 0);
    return (height - current) * Number((base.blocktime / 1000).toFixed()) * 1000;
  }
  const now = new Date();
  const end = new Date(proposal.value.content?.plan?.time || '');
  return end.getTime() - now.getTime();
});

const total = computed(() => {
  const tally = proposal.value.final_tally_result;
  let sum = 0;
  if (tally) {
    sum += Number(tally.abstain || tally.abstain_count || 0);
    sum += Number(tally.yes || tally.yes_count || 0);
    sum += Number(tally.no || tally.no_count || 0);
    sum += Number(tally.no_with_veto || tally.no_with_veto_count || 0);
  }
  return sum;
});

const turnout = computed(() => {
  if (total.value > 0) {
    const bonded = stakingStore.pool?.bonded_tokens || '1';
    return format.percent(total.value / Number(bonded));
  }
  return '0%';
});

const yes = computed(() => {
  if (total.value > 0) {
    const tally = proposal.value?.final_tally_result;
    const yesVal = tally?.yes || tally?.yes_count || 0;
    return format.percent(Number(yesVal) / total.value);
  }
  return '0%';
});

const no = computed(() => {
  if (total.value > 0) {
    const tally = proposal.value?.final_tally_result;
    const value = tally?.no || tally?.no_count || 0;
    return format.percent(Number(value) / total.value);
  }
  return '0%';
});

const veto = computed(() => {
  if (total.value > 0) {
    const tally = proposal.value?.final_tally_result;
    const value = tally?.no_with_veto || tally?.no_with_veto_count || 0;
    return format.percent(Number(value) / total.value);
  }
  return '0%';
});

const abstain = computed(() => {
  if (total.value > 0) {
    const tally = proposal.value?.final_tally_result;
    const value = tally?.abstain || tally?.abstain_count || 0;
    return format.percent(Number(value) / total.value);
  }
  return '0%';
});

const yesRaw = computed(() => {
  const tally = proposal.value?.final_tally_result;
  return Number(tally?.yes || tally?.yes_count || 0);
});
const noRaw = computed(() => {
  const tally = proposal.value?.final_tally_result;
  return Number(tally?.no || tally?.no_count || 0);
});
const vetoRaw = computed(() => {
  const tally = proposal.value?.final_tally_result;
  return Number(tally?.no_with_veto || tally?.no_with_veto_count || 0);
});
const abstainRaw = computed(() => {
  const tally = proposal.value?.final_tally_result;
  return Number(tally?.abstain || tally?.abstain_count || 0);
});

const tallyItems = computed(() => [
  { 
    name: 'Yes', 
    value: yes.value, 
    raw: yesRaw.value,
    icon: 'mdi:check-circle',
    color: 'emerald',
    bgClass: 'from-emerald-500 to-green-400'
  },
  { 
    name: 'No', 
    value: no.value, 
    raw: noRaw.value,
    icon: 'mdi:close-circle',
    color: 'red',
    bgClass: 'from-red-500 to-rose-400'
  },
  { 
    name: 'No With Veto', 
    value: veto.value, 
    raw: vetoRaw.value,
    icon: 'mdi:cancel',
    color: 'orange',
    bgClass: 'from-orange-500 to-amber-400'
  },
  { 
    name: 'Abstain', 
    value: abstain.value, 
    raw: abstainRaw.value,
    icon: 'mdi:minus-circle',
    color: 'gray',
    bgClass: 'from-gray-500 to-slate-400'
  },
]);

function getVoterInfo(voter: string) {
  try {
    const { data } = fromBech32(voter);
    const hex = toHex(data);
    const v = stakingStore.validators.find((x) => toHex(fromBech32(x.operator_address).data) === hex);
    return v ? { name: v.description.moniker, isValidator: true, validator: v } : { name: voter, isValidator: false };
  } catch (e) {
    return { name: voter, isValidator: false };
  }
}

function showValidatorName(voter: string) {
  return getVoterInfo(voter).name;
}

const validatorVotes = computed(() => {
  const validators = stakingStore.validators || [];
  const voteMap = new Map<string, string>();
  
  if (votes.value && Array.isArray(votes.value)) {
    votes.value.forEach((v) => {
      try {
        const { data } = fromBech32(v.voter);
        const hex = toHex(data);
        const option = v.option || (v.options && v.options[0]?.option) || 'VOTE_OPTION_UNSPECIFIED';
        voteMap.set(hex, option);
      } catch (e) {}
    });
  }

  const result = {
    yes: [] as any[],
    no: [] as any[],
    abstain: [] as any[],
    veto: [] as any[],
    notVoted: [] as any[]
  };

  validators.forEach((val) => {
    try {
      const { data } = fromBech32(val.operator_address);
      const hex = toHex(data);
      const vote = voteMap.get(hex);
      const validatorInfo = {
        moniker: val.description?.moniker || 'Unknown',
        operatorAddress: val.operator_address,
        votingPower: val.tokens,
        vote: vote
      };

      if (!vote || vote === 'VOTE_OPTION_UNSPECIFIED') {
        result.notVoted.push(validatorInfo);
      } else if (vote === 'VOTE_OPTION_YES') {
        result.yes.push(validatorInfo);
      } else if (vote === 'VOTE_OPTION_NO') {
        result.no.push(validatorInfo);
      } else if (vote === 'VOTE_OPTION_ABSTAIN') {
        result.abstain.push(validatorInfo);
      } else if (vote === 'VOTE_OPTION_NO_WITH_VETO') {
        result.veto.push(validatorInfo);
      }
    } catch (e) {}
  });

  return result;
});

const userVotes = computed(() => {
  if (!votes.value || !Array.isArray(votes.value)) return [];
  
  return votes.value.filter((v) => {
    const info = getVoterInfo(v.voter);
    return !info.isValidator;
  }).map((v) => ({
    voter: v.voter,
    option: v.option || (v.options && v.options[0]?.option) || 'VOTE_OPTION_UNSPECIFIED',
    options: v.options
  }));
});

function pageload(p: number) {
  pageRequest.value.setPage(p);
  store.fetchProposalVotes(props.proposal_id, pageRequest.value).then((x) => {
    votes.value = x.votes;
    pageResponse.value = x.pagination;
  });
}

function metaItem(metadata: string | undefined): { title: string; summary: string } {
  if (!metadata) {
    return { title: '', summary: '' };
  } else if (metadata.startsWith('{') && metadata.endsWith('}')) {
    return JSON.parse(metadata);
  }
  return { title: metadata, summary: '' };
}

function formatCountdown(ms: number) {
  if (ms <= 0) return { days: 0, hours: 0, minutes: 0, seconds: 0, expired: true };
  
  const seconds = Math.floor(ms / 1000);
  const minutes = Math.floor(seconds / 60);
  const hours = Math.floor(minutes / 60);
  const days = Math.floor(hours / 24);
  
  return {
    days,
    hours: hours % 24,
    minutes: minutes % 60,
    seconds: seconds % 60,
    expired: false
  };
}

const countdownDisplay = computed(() => formatCountdown(votingCountdown.value));

const liveCountdown = computed(() => {
  const end = new Date(proposal.value.voting_end_time);
  const now = new Date();
  const diff = end.getTime() - now.getTime();
  return formatCountdown(diff);
});

function getVoteOptionClass(option: string) {
  switch(option) {
    case 'VOTE_OPTION_YES': return 'bg-emerald-100 dark:bg-emerald-500/20 text-emerald-600 dark:text-emerald-400';
    case 'VOTE_OPTION_NO': return 'bg-red-100 dark:bg-red-500/20 text-red-600 dark:text-red-400';
    case 'VOTE_OPTION_ABSTAIN': return 'bg-gray-100 dark:bg-gray-500/20 text-gray-600 dark:text-gray-400';
    case 'VOTE_OPTION_NO_WITH_VETO': return 'bg-orange-100 dark:bg-orange-500/20 text-orange-600 dark:text-orange-400';
    default: return 'bg-gray-100 dark:bg-gray-500/20 text-gray-500';
  }
}

function formatVoteOption(option: string) {
  return option.replace('VOTE_OPTION_', '').replace(/_/g, ' ');
}
</script>

<template>
  <div class="space-y-6">
    <div class="rounded-2xl bg-gradient-to-br from-gray-800 via-gray-900 to-gray-800 dark:from-gray-800 dark:via-gray-900 dark:to-gray-800 p-6 md:p-8 border border-gray-700/50 shadow-xl">
      <div class="flex flex-col md:flex-row md:items-start md:justify-between gap-4">
        <div class="flex-1">
          <div class="flex items-center gap-3 mb-3">
            <span class="px-3 py-1 rounded-full bg-primary/20 text-primary font-semibold text-sm">
              #{{ proposal_id }}
            </span>
            <span 
              class="px-3 py-1 rounded-full text-sm font-medium inline-flex items-center gap-1.5"
              :class="
                color === 'success' ? 'bg-emerald-500/20 text-emerald-400' : 
                color === 'error' ? 'bg-red-500/20 text-red-400' : 
                'bg-blue-500/20 text-blue-400'
              "
            >
              <span class="w-2 h-2 rounded-full" :class="color === 'success' ? 'bg-emerald-400' : color === 'error' ? 'bg-red-400' : 'bg-blue-400 animate-pulse'"></span>
              {{ status }}
            </span>
          </div>
          <h1 class="text-2xl md:text-3xl font-bold text-white mb-2">
            {{ proposalTitle }}
          </h1>
          <p class="text-gray-400 text-sm">
            Submitted {{ format.toDay(proposal.submit_time, 'from') }}
          </p>
        </div>
        
        <div class="flex gap-2">
          <button
            class="px-5 py-2.5 rounded-xl bg-gradient-to-r from-primary to-emerald-500 text-white font-semibold text-sm hover:opacity-90 transition-all shadow-lg shadow-primary/20"
            @click="dialog.open('vote', { proposal_id })"
          >
            <Icon icon="mdi:vote" class="inline mr-1.5" />
            Vote
          </button>
          <button
            class="px-5 py-2.5 rounded-xl bg-gray-700 hover:bg-gray-600 text-white font-semibold text-sm transition-all"
            @click="dialog.open('deposit', { proposal_id })"
          >
            <Icon icon="mdi:cash-plus" class="inline mr-1.5" />
            Deposit
          </button>
        </div>
      </div>
    </div>

    <div class="grid lg:grid-cols-2 gap-6">
      <div class="rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200 dark:border-gray-700/50 p-6 shadow-sm">
        <div class="flex items-center gap-3 mb-6">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-br from-blue-500 to-indigo-600 flex items-center justify-center">
            <Icon icon="mdi:chart-pie" class="text-white text-xl" />
          </div>
          <h2 class="text-xl font-bold text-gray-900 dark:text-white">Tally Results</h2>
        </div>

        <div class="mb-6">
          <div class="flex items-center justify-between mb-2">
            <span class="text-sm text-gray-500 dark:text-gray-400">Turnout</span>
            <span class="text-sm font-semibold text-gray-900 dark:text-white">{{ turnout }}</span>
          </div>
          <div class="h-3 bg-gray-100 dark:bg-gray-700 rounded-full overflow-hidden">
            <div 
              class="h-full bg-gradient-to-r from-blue-500 to-indigo-500 rounded-full transition-all duration-500"
              :style="`width: ${turnout === '-' || turnout === 'NaN%' ? '0%' : turnout}`"
            ></div>
          </div>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <div 
            v-for="item in tallyItems" 
            :key="item.name"
            class="relative p-4 rounded-xl bg-gray-50 dark:bg-gray-700/50 border border-gray-100 dark:border-gray-600/30 overflow-hidden"
          >
            <div class="absolute top-0 right-0 w-16 h-16 opacity-10">
              <Icon :icon="item.icon" class="w-full h-full" :class="`text-${item.color}-500`" />
            </div>
            <div class="relative">
              <div class="flex items-center gap-2 mb-2">
                <div 
                  class="w-8 h-8 rounded-lg bg-gradient-to-br flex items-center justify-center"
                  :class="item.bgClass"
                >
                  <Icon :icon="item.icon" class="text-white text-sm" />
                </div>
                <span class="text-sm font-medium text-gray-600 dark:text-gray-300">{{ item.name }}</span>
              </div>
              <div class="text-2xl font-bold text-gray-900 dark:text-white">
                {{ item.value === '-' || item.value === 'NaN%' ? '0%' : item.value }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200 dark:border-gray-700/50 p-6 shadow-sm">
        <div class="flex items-center gap-3 mb-6">
          <div class="w-10 h-10 rounded-xl bg-gradient-to-br from-purple-500 to-pink-600 flex items-center justify-center">
            <Icon icon="mdi:timeline-clock" class="text-white text-xl" />
          </div>
          <h2 class="text-xl font-bold text-gray-900 dark:text-white">Timeline</h2>
        </div>

        <div v-if="status === 'VOTING_PERIOD'" class="mb-6 p-4 rounded-xl bg-gradient-to-r from-blue-500/10 to-indigo-500/10 border border-blue-500/20">
          <div class="text-sm text-gray-500 dark:text-gray-400 mb-2">Voting ends in</div>
          <div class="flex items-center gap-3">
            <div class="text-center">
              <div class="text-3xl font-bold text-gray-900 dark:text-white">{{ liveCountdown.days }}</div>
              <div class="text-xs text-gray-500 dark:text-gray-400 uppercase">Days</div>
            </div>
            <span class="text-2xl text-gray-400">:</span>
            <div class="text-center">
              <div class="text-3xl font-bold text-gray-900 dark:text-white">{{ String(liveCountdown.hours).padStart(2, '0') }}</div>
              <div class="text-xs text-gray-500 dark:text-gray-400 uppercase">Hours</div>
            </div>
            <span class="text-2xl text-gray-400">:</span>
            <div class="text-center">
              <div class="text-3xl font-bold text-gray-900 dark:text-white">{{ String(liveCountdown.minutes).padStart(2, '0') }}</div>
              <div class="text-xs text-gray-500 dark:text-gray-400 uppercase">Mins</div>
            </div>
            <span class="text-2xl text-gray-400">:</span>
            <div class="text-center">
              <div class="text-3xl font-bold text-blue-500">{{ String(liveCountdown.seconds).padStart(2, '0') }}</div>
              <div class="text-xs text-gray-500 dark:text-gray-400 uppercase">Secs</div>
            </div>
          </div>
        </div>

        <div class="relative pl-6">
          <div class="absolute left-2 top-2 bottom-2 w-0.5 bg-gradient-to-b from-red-500 via-blue-500 to-emerald-500"></div>
          
          <div class="relative pb-6">
            <div class="absolute left-[-18px] w-4 h-4 rounded-full bg-red-500 border-4 border-white dark:border-gray-800"></div>
            <div class="pl-4">
              <div class="text-sm font-medium text-gray-900 dark:text-white">Submitted</div>
              <div class="text-sm text-gray-500 dark:text-gray-400">{{ format.toDay(proposal.submit_time) }}</div>
              <div class="text-xs text-gray-400 mt-1">{{ shortTime(proposal.submit_time) }}</div>
            </div>
          </div>

          <div class="relative pb-6">
            <div class="absolute left-[-18px] w-4 h-4 rounded-full bg-blue-500 border-4 border-white dark:border-gray-800"></div>
            <div class="pl-4">
              <div class="text-sm font-medium text-gray-900 dark:text-white">Deposit Period End</div>
              <div class="text-sm text-gray-500 dark:text-gray-400">
                {{ format.toDay(proposal.status === 'PROPOSAL_STATUS_DEPOSIT_PERIOD' ? proposal.deposit_end_time : proposal.voting_start_time) }}
              </div>
            </div>
          </div>

          <div class="relative pb-6">
            <div class="absolute left-[-18px] w-4 h-4 rounded-full bg-indigo-500 border-4 border-white dark:border-gray-800"></div>
            <div class="pl-4">
              <div class="text-sm font-medium text-gray-900 dark:text-white">Voting Started</div>
              <div class="text-sm text-gray-500 dark:text-gray-400">{{ format.toDay(proposal.voting_start_time) }}</div>
              <div class="text-xs text-gray-400 mt-1">{{ shortTime(proposal.voting_start_time) }}</div>
            </div>
          </div>

          <div class="relative">
            <div class="absolute left-[-18px] w-4 h-4 rounded-full bg-emerald-500 border-4 border-white dark:border-gray-800"></div>
            <div class="pl-4">
              <div class="text-sm font-medium text-gray-900 dark:text-white">Voting End</div>
              <div class="text-sm text-gray-500 dark:text-gray-400">{{ format.toDay(proposal.voting_end_time) }}</div>
              <div class="text-xs text-gray-400 mt-1">{{ shortTime(proposal.voting_end_time) }}</div>
            </div>
          </div>

          <div v-if="proposal?.content?.['@type']?.endsWith('SoftwareUpgradeProposal')" class="relative mt-6">
            <div class="absolute left-[-18px] w-4 h-4 rounded-full bg-amber-500 border-4 border-white dark:border-gray-800"></div>
            <div class="pl-4">
              <div class="text-sm font-medium text-gray-900 dark:text-white">Upgrade Plan</div>
              <div class="text-sm text-gray-500 dark:text-gray-400">
                <span v-if="Number(proposal.content?.plan?.height || '0') > 0">Height: {{ proposal.content?.plan?.height }}</span>
                <span v-else>{{ format.toDay(proposal.content?.plan?.time) }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="proposal.content?.description || proposal.summary || metaItem(proposal?.metadata)?.summary" 
         class="rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200 dark:border-gray-700/50 p-6 shadow-sm">
      <div class="flex items-center gap-3 mb-4">
        <div class="w-10 h-10 rounded-xl bg-gradient-to-br from-gray-500 to-slate-600 flex items-center justify-center">
          <Icon icon="mdi:file-document" class="text-white text-xl" />
        </div>
        <h2 class="text-xl font-bold text-gray-900 dark:text-white">Description</h2>
      </div>
      <div class="prose dark:prose-invert max-w-none">
        <ObjectElement :value="proposal.content" />
        <MdEditor
          v-if="(proposal.summary && !proposal.content?.description) || metaItem(proposal?.metadata)?.summary"
          :model-value="format.multiLine(proposal.summary || metaItem(proposal?.metadata)?.summary)"
          previewOnly
          class="md-editor-recover"
        />
      </div>
    </div>

    <div class="rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200 dark:border-gray-700/50 shadow-sm overflow-hidden">
      <div class="p-6 border-b border-gray-200 dark:border-gray-700/50">
        <div class="flex items-center justify-between">
          <div class="flex items-center gap-3">
            <div class="w-10 h-10 rounded-xl bg-gradient-to-br from-amber-500 to-orange-600 flex items-center justify-center">
              <Icon icon="mdi:vote" class="text-white text-xl" />
            </div>
            <h2 class="text-xl font-bold text-gray-900 dark:text-white">Votes</h2>
          </div>
          
          <div class="flex bg-gray-100 dark:bg-gray-700/50 rounded-xl p-1">
            <button
              @click="votesTab = 'validators'"
              class="px-4 py-2 rounded-lg text-sm font-medium transition-all"
              :class="votesTab === 'validators' 
                ? 'bg-white dark:bg-gray-600 text-gray-900 dark:text-white shadow-sm' 
                : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300'"
            >
              <Icon icon="mdi:shield-check" class="inline mr-1" />
              Validators
            </button>
            <button
              @click="votesTab = 'users'"
              class="px-4 py-2 rounded-lg text-sm font-medium transition-all"
              :class="votesTab === 'users' 
                ? 'bg-white dark:bg-gray-600 text-gray-900 dark:text-white shadow-sm' 
                : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300'"
            >
              <Icon icon="mdi:account-group" class="inline mr-1" />
              Users
            </button>
          </div>
        </div>
      </div>

      <div v-if="votesTab === 'validators'" class="p-6">
        <div class="grid md:grid-cols-3 gap-4 mb-6">
          <div class="p-4 rounded-xl bg-emerald-50 dark:bg-emerald-500/10 border border-emerald-200 dark:border-emerald-500/20">
            <div class="flex items-center gap-2 mb-2">
              <Icon icon="mdi:check-circle" class="text-emerald-500 text-xl" />
              <span class="text-sm font-medium text-emerald-700 dark:text-emerald-400">Yes</span>
            </div>
            <div class="text-2xl font-bold text-emerald-600 dark:text-emerald-400">{{ validatorVotes.yes.length }}</div>
          </div>
          <div class="p-4 rounded-xl bg-red-50 dark:bg-red-500/10 border border-red-200 dark:border-red-500/20">
            <div class="flex items-center gap-2 mb-2">
              <Icon icon="mdi:close-circle" class="text-red-500 text-xl" />
              <span class="text-sm font-medium text-red-700 dark:text-red-400">No</span>
            </div>
            <div class="text-2xl font-bold text-red-600 dark:text-red-400">{{ validatorVotes.no.length + validatorVotes.veto.length }}</div>
          </div>
          <div class="p-4 rounded-xl bg-gray-50 dark:bg-gray-700/30 border border-gray-200 dark:border-gray-600/30">
            <div class="flex items-center gap-2 mb-2">
              <Icon icon="mdi:clock-outline" class="text-gray-500 text-xl" />
              <span class="text-sm font-medium text-gray-700 dark:text-gray-400">Not Voted</span>
            </div>
            <div class="text-2xl font-bold text-gray-600 dark:text-gray-400">{{ validatorVotes.notVoted.length }}</div>
          </div>
        </div>

        <div class="space-y-4">
          <div v-if="validatorVotes.yes.length > 0">
            <h3 class="text-sm font-semibold text-emerald-600 dark:text-emerald-400 mb-3 flex items-center gap-2">
              <Icon icon="mdi:check-circle" />
              Voted Yes ({{ validatorVotes.yes.length }})
            </h3>
            <div class="flex flex-wrap gap-2">
              <span 
                v-for="v in validatorVotes.yes" 
                :key="v.operatorAddress"
                class="px-3 py-1.5 rounded-lg bg-emerald-100 dark:bg-emerald-500/20 text-emerald-700 dark:text-emerald-400 text-sm font-medium"
              >
                {{ v.moniker }}
              </span>
            </div>
          </div>

          <div v-if="validatorVotes.no.length > 0">
            <h3 class="text-sm font-semibold text-red-600 dark:text-red-400 mb-3 flex items-center gap-2">
              <Icon icon="mdi:close-circle" />
              Voted No ({{ validatorVotes.no.length }})
            </h3>
            <div class="flex flex-wrap gap-2">
              <span 
                v-for="v in validatorVotes.no" 
                :key="v.operatorAddress"
                class="px-3 py-1.5 rounded-lg bg-red-100 dark:bg-red-500/20 text-red-700 dark:text-red-400 text-sm font-medium"
              >
                {{ v.moniker }}
              </span>
            </div>
          </div>

          <div v-if="validatorVotes.veto.length > 0">
            <h3 class="text-sm font-semibold text-orange-600 dark:text-orange-400 mb-3 flex items-center gap-2">
              <Icon icon="mdi:cancel" />
              Voted No With Veto ({{ validatorVotes.veto.length }})
            </h3>
            <div class="flex flex-wrap gap-2">
              <span 
                v-for="v in validatorVotes.veto" 
                :key="v.operatorAddress"
                class="px-3 py-1.5 rounded-lg bg-orange-100 dark:bg-orange-500/20 text-orange-700 dark:text-orange-400 text-sm font-medium"
              >
                {{ v.moniker }}
              </span>
            </div>
          </div>

          <div v-if="validatorVotes.abstain.length > 0">
            <h3 class="text-sm font-semibold text-gray-600 dark:text-gray-400 mb-3 flex items-center gap-2">
              <Icon icon="mdi:minus-circle" />
              Abstained ({{ validatorVotes.abstain.length }})
            </h3>
            <div class="flex flex-wrap gap-2">
              <span 
                v-for="v in validatorVotes.abstain" 
                :key="v.operatorAddress"
                class="px-3 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-700/50 text-gray-700 dark:text-gray-400 text-sm font-medium"
              >
                {{ v.moniker }}
              </span>
            </div>
          </div>

          <div v-if="validatorVotes.notVoted.length > 0">
            <h3 class="text-sm font-semibold text-gray-500 mb-3 flex items-center gap-2">
              <Icon icon="mdi:clock-outline" />
              Not Voted Yet ({{ validatorVotes.notVoted.length }})
            </h3>
            <div class="flex flex-wrap gap-2">
              <span 
                v-for="v in validatorVotes.notVoted.slice(0, 20)" 
                :key="v.operatorAddress"
                class="px-3 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-700/30 text-gray-500 dark:text-gray-500 text-sm"
              >
                {{ v.moniker }}
              </span>
              <span 
                v-if="validatorVotes.notVoted.length > 20"
                class="px-3 py-1.5 rounded-lg bg-gray-100 dark:bg-gray-700/30 text-gray-500 text-sm"
              >
                +{{ validatorVotes.notVoted.length - 20 }} more
              </span>
            </div>
          </div>
        </div>
      </div>

      <div v-if="votesTab === 'users'" class="p-6">
        <div v-if="userVotes.length === 0" class="text-center py-8">
          <Icon icon="mdi:account-question" class="text-4xl text-gray-400 mb-2" />
          <p class="text-gray-500 dark:text-gray-400">No user votes found</p>
        </div>
        
        <div v-else class="space-y-2">
          <div 
            v-for="(vote, index) in userVotes" 
            :key="index"
            class="flex items-center justify-between p-3 rounded-xl bg-gray-50 dark:bg-gray-700/30 border border-gray-100 dark:border-gray-600/30"
          >
            <div class="flex-1 min-w-0">
              <p class="text-sm font-medium text-gray-900 dark:text-white truncate">
                {{ vote.voter }}
              </p>
            </div>
            <span 
              class="ml-3 px-3 py-1 rounded-full text-xs font-medium"
              :class="getVoteOptionClass(vote.option)"
            >
              {{ formatVoteOption(vote.option) }}
            </span>
          </div>
        </div>

        <PaginationBar 
          v-if="pageResponse.total" 
          :limit="pageRequest.limit" 
          :total="pageResponse.total" 
          :callback="pageload" 
          class="mt-4"
        />
      </div>
    </div>
  </div>
</template>
