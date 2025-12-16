<script lang="ts" setup>
import { useGovStore } from '@/stores';
import ProposalListItem from '@/components/ProposalListItem.vue';
import { ref, onMounted, computed } from 'vue';
import PaginationBar from '@/components/PaginationBar.vue';
import { PageRequest } from '@/types';
import { Icon } from '@iconify/vue';

const tab = ref('2');
const store = useGovStore();
const pageRequest = ref(new PageRequest());

onMounted(() => {
  store.fetchProposals('2').then((x) => {
    if (x?.proposals?.length === 0) {
      tab.value = '3';
      store.fetchProposals('3');
    }
    store.fetchProposals('3');
    store.fetchProposals('4');
  });
});

const changeTab = (val: '2' | '3' | '4') => {
  tab.value = val;
};

function page(p: number) {
  pageRequest.value.setPage(p);
  store.fetchProposals(tab.value, pageRequest.value);
}

const proposalCount = computed(() => store?.proposals[tab.value]?.proposals?.length || 0);
</script>
<template>
  <div class="space-y-6">
    <!-- Page Header -->
    <div class="relative overflow-hidden rounded-2xl bg-gradient-to-br from-gray-50 via-white to-gray-100 dark:from-slate-800 dark:via-slate-800 dark:to-slate-900 border border-gray-200 dark:border-gray-700/50 p-6 lg:p-8">
      <div class="absolute top-0 right-0 w-64 h-64 bg-gradient-to-br from-orange-500/10 to-red-500/10 rounded-full blur-3xl"></div>
      <div class="absolute bottom-0 left-0 w-48 h-48 bg-gradient-to-br from-primary/10 to-emerald-500/10 rounded-full blur-3xl"></div>
      
      <div class="relative flex flex-col md:flex-row md:items-center md:justify-between gap-4">
        <div class="flex items-center gap-4">
          <div class="p-3 rounded-xl bg-gradient-to-br from-orange-500 to-red-500 shadow-lg shadow-orange-500/20">
            <Icon icon="mdi:vote" class="text-3xl text-white" />
          </div>
          <div>
            <h1 class="text-2xl lg:text-3xl font-bold text-gray-900 dark:text-white">Governance</h1>
            <p class="text-gray-500 dark:text-gray-400">Participate in network governance decisions</p>
          </div>
        </div>
        
        <div class="flex items-center gap-2 text-sm">
          <span class="px-3 py-1.5 rounded-full bg-primary/10 text-primary font-medium">
            {{ proposalCount }} {{ proposalCount === 1 ? 'Proposal' : 'Proposals' }}
          </span>
        </div>
      </div>
    </div>

    <!-- Tab Navigation -->
    <div class="flex flex-wrap gap-2 p-1.5 rounded-2xl bg-gray-100 dark:bg-gray-800/50 border border-gray-200 dark:border-gray-700/50">
      <button 
        @click="changeTab('2')"
        class="flex-1 flex items-center justify-center gap-2 px-4 py-3 rounded-xl font-medium text-sm transition-all"
        :class="tab === '2' 
          ? 'bg-white dark:bg-gray-700 text-primary shadow-sm' 
          : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300 hover:bg-white/50 dark:hover:bg-gray-700/50'"
      >
        <Icon icon="mdi:vote-outline" class="text-lg" />
        <span>{{ $t('gov.voting') }}</span>
        <span v-if="store?.proposals['2']?.proposals?.length" class="px-1.5 py-0.5 rounded-full text-xs bg-blue-500/10 text-blue-600 dark:text-blue-400">
          {{ store?.proposals['2']?.proposals?.length }}
        </span>
      </button>
      
      <button 
        @click="changeTab('3')"
        class="flex-1 flex items-center justify-center gap-2 px-4 py-3 rounded-xl font-medium text-sm transition-all"
        :class="tab === '3' 
          ? 'bg-white dark:bg-gray-700 text-emerald-600 dark:text-emerald-400 shadow-sm' 
          : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300 hover:bg-white/50 dark:hover:bg-gray-700/50'"
      >
        <Icon icon="mdi:check-circle-outline" class="text-lg" />
        <span>{{ $t('gov.passed') }}</span>
        <span v-if="store?.proposals['3']?.proposals?.length" class="px-1.5 py-0.5 rounded-full text-xs bg-emerald-500/10 text-emerald-600 dark:text-emerald-400">
          {{ store?.proposals['3']?.proposals?.length }}
        </span>
      </button>
      
      <button 
        @click="changeTab('4')"
        class="flex-1 flex items-center justify-center gap-2 px-4 py-3 rounded-xl font-medium text-sm transition-all"
        :class="tab === '4' 
          ? 'bg-white dark:bg-gray-700 text-red-600 dark:text-red-400 shadow-sm' 
          : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300 hover:bg-white/50 dark:hover:bg-gray-700/50'"
      >
        <Icon icon="mdi:close-circle-outline" class="text-lg" />
        <span>{{ $t('gov.rejected') }}</span>
        <span v-if="store?.proposals['4']?.proposals?.length" class="px-1.5 py-0.5 rounded-full text-xs bg-red-500/10 text-red-600 dark:text-red-400">
          {{ store?.proposals['4']?.proposals?.length }}
        </span>
      </button>
    </div>

    <!-- Proposals List -->
    <div class="rounded-2xl bg-white dark:bg-gray-800/80 border border-gray-200 dark:border-gray-700/50 overflow-hidden shadow-sm min-h-[100px]">
      <ProposalListItem :proposals="store?.proposals[tab]" />
      
      <!-- Empty State -->
      <div v-if="!store?.proposals[tab]?.proposals?.length" class="py-16 flex flex-col items-center justify-center text-center">
        <div class="p-4 rounded-full bg-gray-100 dark:bg-gray-700/50 mb-4">
          <Icon 
            :icon="tab === '2' ? 'mdi:vote-outline' : tab === '3' ? 'mdi:check-circle-outline' : 'mdi:close-circle-outline'" 
            class="text-5xl text-gray-400"
          />
        </div>
        <h3 class="text-lg font-semibold text-gray-900 dark:text-white mb-1">No proposals found</h3>
        <p class="text-gray-500 dark:text-gray-400">
          {{ tab === '2' ? 'There are no active voting proposals' : tab === '3' ? 'No passed proposals yet' : 'No rejected proposals' }}
        </p>
      </div>
    </div>

    <!-- Pagination -->
    <PaginationBar :total="store?.proposals[tab]?.pagination?.total" :limit="pageRequest.limit" :callback="page" />
  </div>
</template>
<route>
  {
    meta: {
      i18n: 'governance',
      order: 2
    }
  }
</route>
