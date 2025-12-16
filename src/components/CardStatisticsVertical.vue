<script setup lang="ts">
import { Icon } from '@iconify/vue';
import { controlledComputed } from '@vueuse/core';

interface Props {
  title: string;
  color?: string;
  icon: string;
  stats: string;
  change?: number;
  subtitle?: string;
}

const props = withDefaults(defineProps<Props>(), {
  color: 'primary',
});

const isPositive = controlledComputed(
  () => props.change,
  () => Math.sign(props.change || 0) === 1
);
</script>

<template>
  <div class="group relative overflow-hidden rounded-xl bg-white/60 dark:bg-gray-800/60 backdrop-blur-sm border border-gray-200/50 dark:border-gray-700/50 p-5 hover:border-primary/30 transition-all hover:shadow-lg hover:shadow-primary/5">
    <div class="absolute top-0 right-0 w-24 h-24 bg-gradient-to-br from-primary/5 to-transparent rounded-bl-full"></div>
    
    <div class="relative">
      <div class="flex items-center justify-center mb-3">
        <div v-if="props.icon" class="p-3 rounded-xl bg-gradient-to-br from-primary/10 to-emerald-500/10">
          <Icon :class="[`text-${props?.color}`]" :icon="props.icon" class="text-2xl" />
        </div>
      </div>

      <div
        v-if="props.change"
        :class="isPositive ? 'text-emerald-500' : 'text-red-500'"
        class="flex items-center justify-center text-xs font-semibold mb-1"
      >
        <Icon :icon="isPositive ? 'mdi:trending-up' : 'mdi:trending-down'" class="mr-1" />
        <span>{{ isPositive ? `+${props.change}` : props.change }}%</span>
      </div>

      <h6 class="text-xl text-center font-bold text-gray-900 dark:text-white mb-1">
        {{ props.stats || '-' }}
      </h6>
      <p class="text-xs text-center text-gray-500 dark:text-gray-400 uppercase tracking-wide">
        {{ props.title }}
      </p>

      <div v-if="props.subtitle" class="text-xs text-center text-gray-400 mt-1 font-medium">
        <span class="truncate">{{ props.subtitle }}</span>
      </div>
    </div>
  </div>
</template>
