<script lang="ts" setup>
import { useBlockchain, useFormatter, useStakingStore, useTxDialog } from '@/stores';
import { select } from '@/components/dynamic/index';
import type { PaginatedProposals } from '@/types';
import ProposalProcess from './ProposalProcess.vue';
import type { PropType } from 'vue';
import { computed, ref } from 'vue';
const dialog = useTxDialog();
defineProps({
  proposals: { type: Object as PropType<PaginatedProposals> },
});

const format = useFormatter();
const staking = useStakingStore();
const chain = useBlockchain();
function showType(v: string) {
  if (v) {
    return v.substring(v.lastIndexOf('.') + 1);
  }
  return v;
}

const statusMap: Record<string, string> = {
  PROPOSAL_STATUS_VOTING_PERIOD: 'VOTING',
  PROPOSAL_STATUS_PASSED: 'PASSED',
  PROPOSAL_STATUS_REJECTED: 'REJECTED',
};
const voterStatusMap: Record<string, string> = {
  VOTE_OPTION_NO_WITH_VETO: '',
  VOTE_OPTION_YES: 'success',
  VOTE_OPTION_NO: 'error',
  VOTE_OPTION_ABSTAIN: 'warning',
};

const proposalInfo = ref();

function metaItem(metadata: string | undefined): { title: string; summary: string } {
  return metadata ? JSON.parse(metadata) : {};
}
</script>
<template>
  <div class="text-sm">
    <!-- Desktop View -->
    <div class="hidden lg:block divide-y divide-gray-100 dark:divide-gray-700/50">
      <div 
        v-for="(item, index) in proposals?.proposals" 
        :key="index"
        class="group p-5 hover:bg-gray-50 dark:hover:bg-gray-700/30 transition-all"
      >
        <div class="flex items-start gap-4">
          <!-- Proposal ID Badge -->
          <label
            for="proposal-detail-modal"
            class="flex-none px-3 py-1.5 rounded-lg bg-primary/10 text-primary font-semibold text-sm cursor-pointer hover:bg-primary/20 transition-all"
            @click="proposalInfo = item"
          >
            #{{ item?.proposal_id }}
          </label>

          <!-- Proposal Content -->
          <div class="flex-1 min-w-0">
            <RouterLink
              :to="`/${chain.chainName}/gov/${item?.proposal_id}`"
              class="text-base font-semibold text-gray-900 dark:text-white hover:text-primary transition-colors block truncate"
            >
              {{ item?.content?.title || item?.title || metaItem(item?.metadata)?.title }}
            </RouterLink>
            <div class="flex items-center gap-2 mt-2">
              <span
                v-if="item.content"
                class="px-2.5 py-1 rounded-full text-xs font-medium bg-purple-100 dark:bg-purple-500/20 text-purple-600 dark:text-purple-400"
              >
                {{ showType(item.content['@type']) }}
              </span>
              <span class="text-xs text-gray-500 dark:text-gray-400">
                {{ format.toDay(item.voting_end_time, 'from') }}
              </span>
            </div>
          </div>

          <!-- Progress Bar -->
          <div class="flex-none w-48">
            <ProposalProcess :pool="staking.pool" :tally="item.final_tally_result"></ProposalProcess>
          </div>

          <!-- Status -->
          <div class="flex-none w-28 text-right">
            <span
              class="inline-flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-medium"
              :class="
                statusMap?.[item?.status] === 'PASSED'
                  ? 'bg-emerald-100 dark:bg-emerald-500/20 text-emerald-600 dark:text-emerald-400'
                  : statusMap?.[item?.status] === 'REJECTED'
                  ? 'bg-red-100 dark:bg-red-500/20 text-red-600 dark:text-red-400'
                  : 'bg-blue-100 dark:bg-blue-500/20 text-blue-600 dark:text-blue-400'
              "
            >
              <span
                class="w-1.5 h-1.5 rounded-full"
                :class="
                  statusMap?.[item?.status] === 'PASSED'
                    ? 'bg-emerald-500'
                    : statusMap?.[item?.status] === 'REJECTED'
                    ? 'bg-red-500'
                    : 'bg-blue-500'
                "
              ></span>
              {{ statusMap?.[item?.status] || item?.status }}
            </span>
          </div>

          <!-- Vote Button -->
          <div v-if="statusMap?.[item?.status] === 'VOTING'" class="flex-none">
            <label
              for="vote"
              class="inline-flex items-center gap-1.5 px-4 py-2 rounded-xl bg-gradient-to-r from-primary to-emerald-500 text-white font-medium text-sm cursor-pointer hover:opacity-90 transition-all shadow-lg shadow-primary/20"
              @click="dialog.open('vote', { proposal_id: item?.proposal_id })"
            >
              <span v-if="item?.voterStatus !== 'VOTE_OPTION_NO_WITH_VETO'">
                {{ item?.voterStatus?.replace('VOTE_OPTION_', '') }}
              </span>
              <span v-else>Vote</span>
            </label>
          </div>
        </div>
      </div>
    </div>

    <!-- Mobile View -->
    <div class="lg:hidden divide-y divide-gray-100 dark:divide-gray-700/50">
      <div v-for="(item, index) in proposals?.proposals" :key="index" class="p-4">
        <div class="flex items-start justify-between gap-3 mb-3">
          <RouterLink 
            :to="`/${chain.chainName}/gov/${item?.proposal_id}`" 
            class="flex-1 text-base font-semibold text-gray-900 dark:text-white hover:text-primary truncate"
          >
            {{ item?.content?.title || item?.title || metaItem(item?.metadata)?.title }}
          </RouterLink>
          <label
            for="proposal-detail-modal"
            class="flex-none px-2 py-1 rounded-lg bg-primary/10 text-primary text-sm font-medium cursor-pointer"
            @click="proposalInfo = item"
          >
            #{{ item?.proposal_id }}
          </label>
        </div>

        <div class="flex items-center gap-2 mb-3">
          <span
            v-if="item.content"
            class="px-2 py-0.5 rounded-full text-xs font-medium bg-purple-100 dark:bg-purple-500/20 text-purple-600 dark:text-purple-400"
          >
            {{ showType(item.content['@type']) }}
          </span>
          <span class="text-xs text-gray-500 dark:text-gray-400">
            {{ format.toDay(item.voting_end_time, 'from') }}
          </span>
          <span
            class="ml-auto inline-flex items-center gap-1 px-2 py-0.5 rounded-full text-xs font-medium"
            :class="
              statusMap?.[item?.status] === 'PASSED'
                ? 'bg-emerald-100 dark:bg-emerald-500/20 text-emerald-600 dark:text-emerald-400'
                : statusMap?.[item?.status] === 'REJECTED'
                ? 'bg-red-100 dark:bg-red-500/20 text-red-600 dark:text-red-400'
                : 'bg-blue-100 dark:bg-blue-500/20 text-blue-600 dark:text-blue-400'
            "
          >
            <span
              class="w-1 h-1 rounded-full"
              :class="
                statusMap?.[item?.status] === 'PASSED'
                  ? 'bg-emerald-500'
                  : statusMap?.[item?.status] === 'REJECTED'
                  ? 'bg-red-500'
                  : 'bg-blue-500'
              "
            ></span>
            {{ statusMap?.[item?.status] || item?.status }}
          </span>
        </div>

        <div class="mb-3">
          <ProposalProcess :pool="staking.pool" :tally="item.final_tally_result"></ProposalProcess>
        </div>

        <div v-if="statusMap?.[item?.status] === 'VOTING'" class="mt-3">
          <label
            for="vote"
            class="w-full flex items-center justify-center gap-1.5 px-4 py-2.5 rounded-xl bg-gradient-to-r from-primary to-emerald-500 text-white font-medium text-sm cursor-pointer hover:opacity-90 transition-all"
            @click="dialog.open('vote', { proposal_id: item?.proposal_id })"
          >
            <span v-if="item?.voterStatus !== 'VOTE_OPTION_NO_WITH_VETO'">
              {{ item?.voterStatus?.replace('VOTE_OPTION_', '') }}
            </span>
            <span v-else>Vote</span>
          </label>
        </div>
      </div>
    </div>

    <!-- Modal -->
    <input type="checkbox" id="proposal-detail-modal" class="modal-toggle" />
    <label for="proposal-detail-modal" class="modal">
      <label class="modal-box !w-11/12 !max-w-5xl bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700" for="">
        <label for="proposal-detail-modal" class="btn btn-sm btn-circle absolute right-2 top-2 bg-gray-100 dark:bg-gray-700 border-0 hover:bg-gray-200 dark:hover:bg-gray-600">✕</label>
        <h3 class="font-bold text-lg text-gray-900 dark:text-white">Description</h3>
        <div class="py-4 text-gray-700 dark:text-gray-300">
          <Component
            v-if="proposalInfo?.content?.description || proposalInfo?.summary || metaItem(proposalInfo?.metadata)?.summary"
            :is="select(
              proposalInfo?.content?.description || proposalInfo?.summary || metaItem(proposalInfo?.metadata)?.summary,
              'horizontal'
            )"
            :value="proposalInfo?.content?.description || proposalInfo?.summary || metaItem(proposalInfo?.metadata)?.summary"
          >
          </Component>
        </div>
      </label>
    </label>
  </div>
</template>
