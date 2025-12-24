<script lang="ts" setup>
import Countdown from '@chenfengyuan/vue-countdown';
import { ref, computed } from 'vue';

const props = defineProps({
  time: { type: Number },
  css: { type: String },
});

const formatNumber = (num: number) => {
  return num.toString().padStart(2, '0');
};
</script>

<template>
  <Countdown
    v-if="time"
    :time="time > 0 ? time : 0"
    v-slot="{ days, hours, minutes, seconds }"
  >
    <div class="flex flex-wrap items-center justify-center gap-4 md:gap-6">
      <!-- Days -->
      <div class="relative group">
        <div class="absolute inset-0 bg-gradient-to-r from-indigo-500 via-purple-500 to-pink-500 rounded-2xl blur-lg opacity-50 group-hover:opacity-75 transition-opacity"></div>
        <div class="relative bg-gradient-to-br from-indigo-500 via-purple-500 to-pink-500 rounded-2xl p-1">
          <div class="bg-gray-900 dark:bg-gray-800 rounded-xl px-4 md:px-6 py-3 md:py-4 text-center min-w-[80px] md:min-w-[100px]">
            <div class="text-3xl md:text-5xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-indigo-400 via-purple-400 to-pink-400 tabular-nums">
              {{ formatNumber(days) }}
            </div>
            <div class="text-xs md:text-sm text-gray-400 uppercase tracking-wider mt-1">Days</div>
          </div>
        </div>
      </div>

      <!-- Separator -->
      <div class="text-3xl md:text-4xl font-bold text-gray-400 animate-pulse hidden md:block">:</div>

      <!-- Hours -->
      <div class="relative group">
        <div class="absolute inset-0 bg-gradient-to-r from-cyan-500 to-blue-500 rounded-2xl blur-lg opacity-50 group-hover:opacity-75 transition-opacity"></div>
        <div class="relative bg-gradient-to-br from-cyan-500 to-blue-500 rounded-2xl p-1">
          <div class="bg-gray-900 dark:bg-gray-800 rounded-xl px-4 md:px-6 py-3 md:py-4 text-center min-w-[80px] md:min-w-[100px]">
            <div class="text-3xl md:text-5xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-blue-400 tabular-nums">
              {{ formatNumber(hours) }}
            </div>
            <div class="text-xs md:text-sm text-gray-400 uppercase tracking-wider mt-1">Hours</div>
          </div>
        </div>
      </div>

      <!-- Separator -->
      <div class="text-3xl md:text-4xl font-bold text-gray-400 animate-pulse hidden md:block">:</div>

      <!-- Minutes -->
      <div class="relative group">
        <div class="absolute inset-0 bg-gradient-to-r from-emerald-500 to-green-500 rounded-2xl blur-lg opacity-50 group-hover:opacity-75 transition-opacity"></div>
        <div class="relative bg-gradient-to-br from-emerald-500 to-green-500 rounded-2xl p-1">
          <div class="bg-gray-900 dark:bg-gray-800 rounded-xl px-4 md:px-6 py-3 md:py-4 text-center min-w-[80px] md:min-w-[100px]">
            <div class="text-3xl md:text-5xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-emerald-400 to-green-400 tabular-nums">
              {{ formatNumber(minutes) }}
            </div>
            <div class="text-xs md:text-sm text-gray-400 uppercase tracking-wider mt-1">Minutes</div>
          </div>
        </div>
      </div>

      <!-- Separator -->
      <div class="text-3xl md:text-4xl font-bold text-gray-400 animate-pulse hidden md:block">:</div>

      <!-- Seconds -->
      <div class="relative group">
        <div class="absolute inset-0 bg-gradient-to-r from-amber-500 to-orange-500 rounded-2xl blur-lg opacity-50 group-hover:opacity-75 transition-opacity animate-pulse"></div>
        <div class="relative bg-gradient-to-br from-amber-500 to-orange-500 rounded-2xl p-1">
          <div class="bg-gray-900 dark:bg-gray-800 rounded-xl px-4 md:px-6 py-3 md:py-4 text-center min-w-[80px] md:min-w-[100px]">
            <div class="text-3xl md:text-5xl font-bold text-transparent bg-clip-text bg-gradient-to-r from-amber-400 to-orange-400 tabular-nums transition-all">
              {{ formatNumber(seconds) }}
            </div>
            <div class="text-xs md:text-sm text-gray-400 uppercase tracking-wider mt-1">Seconds</div>
          </div>
        </div>
      </div>
    </div>
  </Countdown>
</template>

<style>
@keyframes pulse-glow {
  0%, 100% {
    opacity: 0.5;
  }
  50% {
    opacity: 0.8;
  }
}
</style>
