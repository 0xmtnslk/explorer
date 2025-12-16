<script setup lang="ts">
import { Icon } from '@iconify/vue';
</script>

<template>
  <footer class="bg-white dark:bg-[#1a1f2e] border-t border-gray-200 dark:border-gray-700/50 mt-auto">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
      <div class="grid grid-cols-1 md:grid-cols-3 gap-8">
        <!-- Brand -->
        <div class="space-y-4">
          <div class="flex items-center gap-3">
            <img src="/logo.webp" alt="CoinHunters" class="w-12 h-12 rounded-full" />
            <div>
              <h3 class="font-bold text-lg text-gray-900 dark:text-white">CoinHunters</h3>
              <p class="text-sm text-gray-500 dark:text-gray-400">Explorer</p>
            </div>
          </div>
          <p class="text-sm text-gray-600 dark:text-gray-400">
            Modern blockchain explorer for Cosmos ecosystem. Track validators, governance, and more.
          </p>
        </div>

        <!-- Quick Links -->
        <div>
          <h4 class="font-semibold text-gray-900 dark:text-white mb-4">Quick Links</h4>
          <ul class="space-y-2">
            <li>
              <a href="https://coinhunterstr.com/" target="_blank" class="text-sm text-gray-600 dark:text-gray-400 hover:text-primary transition-colors flex items-center gap-2">
                <Icon icon="mdi:web" />
                Website
              </a>
            </li>
            <li>
              <RouterLink to="/" class="text-sm text-gray-600 dark:text-gray-400 hover:text-primary transition-colors flex items-center gap-2">
                <Icon icon="mdi:home" />
                Explorer Home
              </RouterLink>
            </li>
          </ul>
        </div>

        <!-- Social Links -->
        <div>
          <h4 class="font-semibold text-gray-900 dark:text-white mb-4">Connect With Us</h4>
          <div class="flex gap-4">
            <a
              href="https://coinhunterstr.com/"
              target="_blank"
              class="p-3 bg-gray-100 dark:bg-gray-800 rounded-xl hover:bg-primary hover:text-white transition-all group"
              title="Website"
            >
              <Icon icon="mdi:web" class="text-xl text-gray-600 dark:text-gray-400 group-hover:text-white" />
            </a>
            <a
              href="https://x.com/CoinHuntersTR"
              target="_blank"
              class="p-3 bg-gray-100 dark:bg-gray-800 rounded-xl hover:bg-primary hover:text-white transition-all group"
              title="X (Twitter)"
            >
              <Icon icon="ri:twitter-x-fill" class="text-xl text-gray-600 dark:text-gray-400 group-hover:text-white" />
            </a>
            <a
              href="https://t.me/CoinHuntersTR"
              target="_blank"
              class="p-3 bg-gray-100 dark:bg-gray-800 rounded-xl hover:bg-primary hover:text-white transition-all group"
              title="Telegram"
            >
              <Icon icon="mdi:telegram" class="text-xl text-gray-600 dark:text-gray-400 group-hover:text-white" />
            </a>
          </div>
        </div>
      </div>

      <!-- Copyright -->
      <div class="mt-8 pt-6 border-t border-gray-200 dark:border-gray-700/50 flex flex-col sm:flex-row justify-between items-center gap-4">
        <p class="text-sm text-gray-500 dark:text-gray-400">
          &copy; {{ new Date().getFullYear() }} CoinHunters. All rights reserved.
        </p>
        <p class="text-xs text-gray-400 dark:text-gray-500">
          Powered by Ping.pub Explorer
        </p>
      </div>
    </div>
  </footer>
</template>
