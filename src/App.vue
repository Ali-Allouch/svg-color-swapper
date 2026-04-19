<script setup lang="ts">
import { ref, watch, toRaw } from "vue";

interface DetectedSVG {
  id: string;
  html: string;
  colors: string[];
  originalColors: string[];
}

const detectedSVGs = ref<DetectedSVG[]>([]);
const selectedSVG = ref<DetectedSVG | null>(null);
const isScanning = ref(false);
const isCopied = ref(false);

const scanForSVGs = async () => {
  isScanning.value = true;
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });

  if (tab?.id) {
    const results = await chrome.scripting.executeScript({
      target: { tabId: tab.id },
      func: () => {
        const rgbToHex = (rgb: string) => {
          if (!rgb || rgb === "none" || rgb === "transparent") return null;
          const match = rgb.match(
            /^rgba?\((\d+),\s*(\d+),\s*(\d+)(?:,\s*(\d+(?:\.\d+)?))?\)$/,
          );
          if (!match) return rgb.startsWith("#") ? rgb.toUpperCase() : null;

          const r = parseInt(match[1]);
          const g = parseInt(match[2]);
          const b = parseInt(match[3]);
          return (
            "#" +
            ((1 << 24) + (r << 16) + (g << 8) + b)
              .toString(16)
              .slice(1)
              .toUpperCase()
          );
        };

        const svgs = Array.from(document.querySelectorAll("svg"));
        return svgs.map((svg, index) => {
          const swapperId = `svg-raw-${index}-${Date.now()}`;
          svg.setAttribute("data-swapper-id", swapperId);

          const colorSet = new Set<string>();
          const clone = svg.cloneNode(true) as SVGElement;

          const originalElements = [
            svg,
            ...Array.from(svg.querySelectorAll("*")),
          ];
          const clonedElements = [
            clone,
            ...Array.from(clone.querySelectorAll("*")),
          ];

          originalElements.forEach((el, i) => {
            const style = window.getComputedStyle(el);

            const getActualColor = (prop: "fill" | "stroke") => {
              let val = style[prop];
              if (val === "currentColor") {
                val = style.color;
              }
              return rgbToHex(val);
            };

            const fill = getActualColor("fill");
            const stroke = getActualColor("stroke");

            if (fill && fill !== "none" && fill !== "transparent") {
              el.setAttribute("data-orig-fill", fill);
              clonedElements[i].setAttribute("data-orig-fill", fill);
              clonedElements[i].setAttribute("fill", fill);
              colorSet.add(fill);
            }

            if (stroke && stroke !== "none" && stroke !== "transparent") {
              el.setAttribute("data-orig-stroke", stroke);
              clonedElements[i].setAttribute("data-orig-stroke", stroke);
              clonedElements[i].setAttribute("stroke", stroke);
              colorSet.add(stroke);
            }

            if (
              clonedElements[i] instanceof HTMLElement ||
              clonedElements[i] instanceof SVGElement
            ) {
              (clonedElements[i] as HTMLElement).style.cssText = "";
            }
          });

          return {
            id: swapperId,
            html: clone.outerHTML,
            colors: Array.from(colorSet),
            originalColors: Array.from(colorSet),
          };
        });
      },
    });
    detectedSVGs.value = results[0].result as DetectedSVG[];
  }
  isScanning.value = false;
};

const selectIcon = (svg: DetectedSVG) => {
  selectedSVG.value = svg;
  isCopied.value = false;
};

const getCleanSVG = (html: string) => {
  const parser = new DOMParser();
  const doc = parser.parseFromString(html, "image/svg+xml");
  const svg = doc.querySelector("svg");
  if (!svg) return html;

  const cleanAttributes = (el: Element) => {
    el.removeAttribute("data-swapper-id");
    el.removeAttribute("data-orig-fill");
    el.removeAttribute("data-orig-stroke");

    const styleFill = (el as HTMLElement).style.fill;
    const styleStroke = (el as HTMLElement).style.stroke;

    if (styleFill) el.setAttribute("fill", styleFill);
    if (styleStroke) el.setAttribute("stroke", styleStroke);
    (el as HTMLElement).removeAttribute("style");
  };

  cleanAttributes(svg);
  svg.querySelectorAll("*").forEach(cleanAttributes);

  return svg.outerHTML;
};

const copyCode = async () => {
  if (!selectedSVG.value) return;
  const cleanCode = getCleanSVG(selectedSVG.value.html);
  await navigator.clipboard.writeText(cleanCode);

  isCopied.value = true;

  setTimeout(() => {
    isCopied.value = false;
  }, 2000);
};

const downloadSVG = () => {
  if (!selectedSVG.value) return;
  const cleanCode = getCleanSVG(selectedSVG.value.html);
  const blob = new Blob([cleanCode], { type: "image/svg+xml" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");

  link.href = url;
  link.download = `swapped-icon-${Date.now()}.svg`;
  link.click();
  URL.revokeObjectURL(url);
};

watch(
  () => selectedSVG.value?.colors,
  async (newColors) => {
    if (!selectedSVG.value || !newColors) return;

    const currentId = selectedSVG.value.id;
    const oldCols = toRaw(selectedSVG.value.originalColors);
    const updatedCols = toRaw(newColors);

    const parser = new DOMParser();
    const doc = parser.parseFromString(selectedSVG.value.html, "image/svg+xml");
    const svgEl = doc.querySelector("svg");

    if (svgEl) {
      const elements = [svgEl, ...Array.from(svgEl.querySelectorAll("*"))];
      elements.forEach((el) => {
        const oFill = el.getAttribute("data-orig-fill");
        const oStroke = el.getAttribute("data-orig-stroke");

        oldCols.forEach((oldCol, idx) => {
          if (oFill === oldCol) el.setAttribute("fill", updatedCols[idx]);
          if (oStroke === oldCol) el.setAttribute("stroke", updatedCols[idx]);
        });
      });
      selectedSVG.value.html = svgEl.outerHTML;
    }

    const [tab] = await chrome.tabs.query({
      active: true,
      currentWindow: true,
    });
    if (!tab?.id) return;

    await chrome.scripting.executeScript({
      target: { tabId: tab.id },
      args: [currentId, oldCols, updatedCols],
      func: (id: string, oldColors: string[], currentColors: string[]) => {
        const svg = document.querySelector(`svg[data-swapper-id="${id}"]`);
        if (!svg) return;

        const elements = [svg, ...Array.from(svg.querySelectorAll("*"))];
        elements.forEach((el) => {
          const oFill = el.getAttribute("data-orig-fill");
          const oStroke = el.getAttribute("data-orig-stroke");

          oldColors.forEach((oldCol, idx) => {
            if (oFill === oldCol)
              (el as HTMLElement).style.fill = currentColors[idx];
            if (oStroke === oldCol)
              (el as HTMLElement).style.stroke = currentColors[idx];
          });
        });
      },
    });
  },
  { deep: true },
);
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
        SVG Color Swapper
      </h1>
      <button
        @click="scanForSVGs"
        :disabled="isScanning"
        class="text-xs font-semibold px-3 py-1.5 bg-blue-600 hover:bg-blue-500 rounded-md transition-all active:scale-95 disabled:opacity-50 cursor-pointer"
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
          @click="selectIcon(svg)"
          class="aspect-square bg-slate-800 rounded-lg p-3 border cursor-pointer flex items-center justify-center group transition-all overflow-hidden"
          :class="
            selectedSVG?.id === svg.id
              ? 'border-blue-500 ring-2 ring-blue-500/20'
              : 'border-slate-700 hover:border-slate-500'
          "
        >
          <div
            class="w-full h-full flex items-center justify-center [&>svg]:max-w-full [&>svg]:max-h-full [&>svg]:w-auto [&>svg]:h-auto [&>svg]:block"
            v-html="svg.html"
          ></div>
        </div>
      </div>
    </main>
    <footer
      v-if="selectedSVG"
      class="p-4 border-t border-slate-800 bg-slate-900 sticky bottom-0 shadow-2xl"
    >
      <div class="flex items-center gap-3 mb-5">
        <div
          class="w-10 h-10 bg-slate-800 rounded-md border border-slate-700 p-1.5 flex items-center justify-center [&>svg]:max-w-full [&>svg]:max-h-full [&>svg]:w-auto [&>svg]:h-auto"
          v-html="selectedSVG.html"
        ></div>
        <div class="flex flex-col">
          <p
            class="text-[10px] uppercase tracking-wider font-bold text-slate-500"
          >
            Selected Asset
          </p>
          <p class="text-xs font-semibold text-blue-400">Edit Colors</p>
        </div>
      </div>

      <div class="flex flex-wrap gap-3">
        <div
          v-for="(color, index) in selectedSVG.colors"
          :key="index"
          class="flex items-center gap-2 bg-slate-800/50 p-1.5 rounded-lg border border-slate-700/50"
        >
          <input
            type="color"
            v-model="selectedSVG.colors[index]"
            class="w-6 h-6 rounded cursor-pointer bg-transparent border-none appearance-none"
          />
          <span class="text-[9px] font-mono text-slate-400">{{ color }}</span>
        </div>
      </div>

      <div class="flex gap-2 mt-6">
        <button
          @click="copyCode"
          :class="[
            'flex-1 flex items-center justify-center gap-2 py-2 text-xs font-bold rounded-lg transition-all border active:scale-95 cursor-pointer',
            isCopied
              ? 'bg-emerald-600/20 border-emerald-500 text-emerald-400'
              : 'bg-slate-800 border-slate-700 text-slate-200 hover:bg-slate-700',
          ]"
        >
          <span v-if="isCopied" class="flex items-center gap-1">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="3"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path d="M20 6 9 17l-5-5" />
            </svg>
            Copied!
          </span>
          <span v-else>Copy Code</span>
        </button>

        <button
          @click="downloadSVG"
          class="flex-1 flex items-center justify-center gap-2 py-2 bg-blue-600 hover:bg-blue-500 text-white text-xs font-bold rounded-lg transition-all shadow-lg shadow-blue-500/20 active:scale-95 cursor-pointer"
        >
          <span>Download SVG</span>
        </button>
      </div>
    </footer>
  </div>
</template>
