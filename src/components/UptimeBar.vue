<script lang="ts" setup>
import { computed, watch, ref } from 'vue';
import type { PropType } from 'vue';

const props = defineProps({
  blocks: { type: Array as PropType<{ height: string; color: string }[]> },
});

const lastBlockHeight = ref('');
const pulseIndex = ref(-1);

watch(() => props.blocks, (newBlocks) => {
  if (newBlocks && newBlocks.length > 0) {
    const latestBlock = newBlocks[newBlocks.length - 1];
    if (latestBlock.height !== lastBlockHeight.value && lastBlockHeight.value !== '') {
      pulseIndex.value = newBlocks.length - 1;
      setTimeout(() => {
        pulseIndex.value = -1;
      }, 1000);
    }
    lastBlockHeight.value = latestBlock.height;
  }
}, { deep: true });

const getBlockStyle = (color: string, index: number) => {
  const isLatest = index === (props.blocks?.length || 0) - 1;
  const isPulsing = index === pulseIndex.value;
  
  let bgColor = '';
  let glowColor = '';
  
  if (color.includes('green')) {
    bgColor = 'bg-gradient-to-br from-emerald-400 to-green-500';
    glowColor = 'shadow-emerald-400/50';
  } else if (color.includes('yellow')) {
    bgColor = 'bg-gradient-to-br from-amber-400 to-yellow-500';
    glowColor = 'shadow-amber-400/50';
  } else if (color.includes('red')) {
    bgColor = 'bg-gradient-to-br from-rose-400 to-red-500';
    glowColor = 'shadow-rose-400/50';
  } else {
    bgColor = 'bg-gradient-to-br from-gray-300 to-gray-400 dark:from-gray-600 dark:to-gray-700';
    glowColor = '';
  }
  
  return {
    bgColor,
    glowColor,
    isLatest,
    isPulsing
  };
};
</script>

<template>
  <div class="flex flex-wrap gap-1">
    <div 
      v-for="(item, index) in blocks" 
      :key="index"
      class="group relative cursor-pointer"
    >
      <div 
        class="w-2.5 h-6 rounded-sm transition-all duration-300 hover:scale-125 hover:z-10"
        :class="[
          getBlockStyle(item.color, index).bgColor,
          getBlockStyle(item.color, index).isLatest ? 'ring-2 ring-white dark:ring-gray-800 shadow-lg ' + getBlockStyle(item.color, index).glowColor : '',
          getBlockStyle(item.color, index).isPulsing ? 'animate-pulse-glow' : ''
        ]"
      >
        <div 
          v-if="getBlockStyle(item.color, index).isPulsing"
          class="absolute inset-0 rounded-sm animate-ping opacity-75"
          :class="getBlockStyle(item.color, index).bgColor"
        ></div>
      </div>
      
      <div class="absolute bottom-full left-1/2 -translate-x-1/2 mb-2 px-2 py-1 bg-gray-900 dark:bg-gray-700 text-white text-xs rounded-lg opacity-0 group-hover:opacity-100 transition-opacity whitespace-nowrap pointer-events-none z-20 shadow-lg">
        <div class="font-medium">#{{ item.height }}</div>
        <div class="text-gray-400 text-[10px]">
          {{ item.color.includes('green') ? 'Committed' : item.color.includes('yellow') ? 'Precommit' : item.color.includes('red') ? 'Missed' : 'Pending' }}
        </div>
        <div class="absolute top-full left-1/2 -translate-x-1/2 border-4 border-transparent border-t-gray-900 dark:border-t-gray-700"></div>
      </div>
    </div>
  </div>
</template>

<style>
@keyframes pulse-glow {
  0%, 100% {
    transform: scale(1);
    box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.7);
  }
  50% {
    transform: scale(1.2);
    box-shadow: 0 0 20px 5px rgba(16, 185, 129, 0.5);
  }
}

.animate-pulse-glow {
  animation: pulse-glow 0.5s ease-in-out 2;
}
</style>
