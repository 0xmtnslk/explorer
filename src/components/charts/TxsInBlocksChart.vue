<script lang="ts" setup>
import ApexCharts from 'vue3-apexcharts';
import { computed } from '@vue/reactivity';
import { useBaseStore } from '@/stores';

const baseStore = useBaseStore();

const isDark = computed(() => {
  return document.documentElement.classList.contains('dark') || 
         document.documentElement.getAttribute('data-theme') === 'dark';
});

const options = computed(() => {
  return {
    chart: {
      type: 'bar',
      height: 200,
      toolbar: {
        show: false,
      },
      animations: {
        enabled: true,
        easing: 'easeinout',
        speed: 800,
        animateGradually: {
          enabled: true,
          delay: 50,
        },
        dynamicAnimation: {
          enabled: true,
          speed: 350,
        },
      },
      background: 'transparent',
      sparkline: {
        enabled: false,
      },
    },
    plotOptions: {
      bar: {
        borderRadius: 6,
        borderRadiusApplication: 'end',
        horizontal: false,
        columnWidth: '70%',
        distributed: false,
      },
    },
    fill: {
      type: 'gradient',
      gradient: {
        shade: 'dark',
        type: 'vertical',
        shadeIntensity: 0.3,
        gradientToColors: ['#EC4899'],
        inverseColors: false,
        opacityFrom: 1,
        opacityTo: 0.8,
        stops: [0, 100],
      },
    },
    dataLabels: {
      enabled: false,
    },
    colors: ['#8B5CF6'],
    grid: {
      show: true,
      borderColor: isDark.value ? '#374151' : '#E5E7EB',
      strokeDashArray: 4,
      xaxis: {
        lines: {
          show: false,
        },
      },
      yaxis: {
        lines: {
          show: true,
        },
      },
      padding: {
        top: 0,
        right: 0,
        bottom: 0,
        left: 10,
      },
    },
    xaxis: {
      labels: {
        show: false,
      },
      axisBorder: {
        show: false,
      },
      axisTicks: {
        show: false,
      },
      categories: baseStore.recents
        .slice(0, 50)
        .map((x) => x.block.header.height),
    },
    yaxis: {
      labels: {
        show: true,
        style: {
          colors: isDark.value ? '#9CA3AF' : '#6B7280',
          fontSize: '11px',
        },
        formatter: function (val: number) {
          return Math.floor(val).toString();
        },
      },
    },
    tooltip: {
      enabled: true,
      theme: isDark.value ? 'dark' : 'light',
      style: {
        fontSize: '12px',
      },
      y: {
        formatter: function (val: number) {
          return val + ' transactions';
        },
      },
      x: {
        show: true,
        formatter: function (val: string) {
          return 'Block #' + val;
        },
      },
      marker: {
        show: true,
      },
    },
    states: {
      hover: {
        filter: {
          type: 'lighten',
          value: 0.1,
        },
      },
      active: {
        filter: {
          type: 'darken',
          value: 0.1,
        },
      },
    },
  };
});

const series = computed(() => {
  return [
    {
      name: 'Transactions',
      data: baseStore.recents?.map((x) => x.block.data.txs.length) || [],
    },
  ];
});
</script>

<template>
  <div class="relative">
    <ApexCharts type="bar" height="200" :options="options" :series="series" />
    
    <!-- Gradient Overlay at Bottom -->
    <div class="absolute bottom-0 left-0 right-0 h-8 bg-gradient-to-t from-white dark:from-gray-800 to-transparent pointer-events-none"></div>
  </div>
</template>
