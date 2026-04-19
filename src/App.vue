<script setup lang="ts">
import { ref } from "vue";

interface DetectedSVG {
  id: string;
  html: string;
}

const detectedSVGs = ref<DetectedSVG[]>([]);
const isScanning = ref(false);

const scanForSVGs = async () => {
  isScanning.value = true;

  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });

  if (tab?.id) {
    const results = await chrome.scripting.executeScript({
      target: { tabId: tab.id },
      func: () => {
        const svgs = Array.from(document.querySelectorAll("svg"));
        return svgs.map((svg, index) => ({
          id: `svg-${index}-${Math.random().toString(36).substr(2, 9)}`,
          html: svg.outerHTML,
        }));
      },
    });

    detectedSVGs.value = results[0].result as DetectedSVG[];
  }

  isScanning.value = false;
};
</script>

<template>
  <div
    class="w-[380px] min-h-[450px] bg-slate-900 text-slate-100 flex flex-col font-sans"
  >
    <header
      class="p-4 border-b border-slate-800 flex justify-between items-center bg-slate-900/50 sticky top-0 backdrop-blur-md"
    >
      <h1
        class="text-lg font-bold bg-gradient-to-r from-blue-400 to-emerald-400 bg-clip-text text-transparent"
      >
        SVG Swapper
      </h1>
      <button
        @click="scanForSVGs"
        :disabled="isScanning"
        class="text-xs font-semibold px-3 py-1.5 bg-blue-600 hover:bg-blue-500 rounded-md transition-all active:scale-95 disabled:opacity-50"
      >
        {{ isScanning ? "Searching..." : "Scan Page" }}
      </button>
    </header>

    <main class="flex-1 p-4">
      <div
        v-if="detectedSVGs.length === 0"
        class="h-full flex flex-col items-center justify-center opacity-40 mt-20"
      >
        <p class="text-sm">No SVGs detected yet.</p>
        <p class="text-[10px]">Open a site with icons and click Scan.</p>
      </div>

      <div v-else class="grid grid-cols-3 gap-3">
        <div
          v-for="svg in detectedSVGs"
          :key="svg.id"
          class="aspect-square bg-slate-800 rounded-lg p-3 border border-slate-700 hover:border-blue-500/50 transition-colors cursor-pointer flex items-center justify-center group"
        >
          <div
            class="w-full h-full [&>svg]:max-w-full [&>svg]:max-h-full [&>svg]:m-auto"
            v-html="svg.html"
          ></div>
        </div>
      </div>
    </main>
  </div>
</template>
