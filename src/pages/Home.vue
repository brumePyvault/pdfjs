<script setup>
import { ref } from "vue";
import * as pdfjsLib from "pdfjs-dist/legacy/build/pdf";

pdfjsLib.GlobalWorkerOptions.workerSrc = `//unpkg.com/pdfjs-dist@${pdfjsLib.version}/build/pdf.worker.min.mjs`;

const userInput = ref("");
const keywords = ref([]);
const files = ref([]);
const results = ref({});
const loading = ref(false);
const noMatches = ref(false);

function handleFileChange(event) {
  results.value = {};
  noMatches.value = false;
  files.value = Array.from(event.target.files);
  if (keywords.value.length > 0) {
    loading.value = true;
    files.value.forEach(readPDF);
  }
}

function handleKeywordSubmit() {
  keywords.value = userInput.value
    .split(",")
    .map((kw) => kw.trim())
    .filter((kw) => kw.length > 0);
  results.value = {};
  noMatches.value = false;
  if (files.value.length > 0 && keywords.value.length > 0) {
    loading.value = true;
    files.value.forEach(readPDF);
  }
}

async function readPDF(file) {
  const reader = new FileReader();
  reader.onload = async () => {
    const typedarray = new Uint8Array(reader.result);
    const pdf = await pdfjsLib.getDocument({ data: typedarray }).promise;

    let fileMatches = [];

    for (let i = 0; i < pdf.numPages; i++) {
      const page = await pdf.getPage(i + 1);
      const textContent = await page.getTextContent();
      const text = textContent.items
        .map((item) => item.str)
        .join(" ")
        .toLowerCase();

      keywords.value.forEach((kw) => {
        if (text.includes(kw.toLowerCase())) {
          fileMatches.push({
            page: i + 1,
            keyword: kw,
          });
        }
      });
    }

    if (fileMatches.length > 0) {
      results.value[file.name] = fileMatches;
    }

    if (
      Object.keys(results.value).length === 0 &&
      files.value.indexOf(file) === files.value.length - 1
    ) {
      noMatches.value = true;
    }

    loading.value = false;
  };
  reader.readAsArrayBuffer(file);
}
</script>

<template>
  <div class="p-6 max-w-3xl mx-auto bg-white min-h-dvh shadow rounded">
    <h1 class="text-2xl font-bold text-gray-800 mb-4">PDF Keyword Search</h1>

    <!-- Keyword input -->
    <div class="mb-4">
      <label class="block text-gray-700 font-medium mb-1">
        Enter keywords (comma-separated):
      </label>
      <input
        v-model="userInput"
        type="text"
        placeholder="e.g. Artificial Intelligence, Machine Learning"
        class="w-full px-3 py-2 border text-black border-gray-300 rounded focus:outline-none focus:ring focus:border-blue-500"
      />
      <button
        @click="handleKeywordSubmit"
        class="mt-2 px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
      >
        Submit Keywords
      </button>
    </div>

    <!-- File input -->
    <div class="mb-4">
      <label class="block text-gray-700 font-medium mb-1"
        >Upload PDF Files:</label
      >
      <input
        type="file"
        multiple
        @change="handleFileChange"
        accept=".pdf"
        class="w-full px-3 py-2 border border-gray-300 text-black rounded focus:outline-none focus:ring focus:border-blue-500"
      />
    </div>

    <!-- Processing message -->
    <div v-if="loading" class="text-blue-600 mb-2">Processing PDFs...</div>

    <!-- No match -->
    <div v-if="noMatches" class="text-red-600 mt-4 font-medium">
      No matches found in any PDF.
    </div>

    <!-- Results -->
    <div v-if="Object.keys(results).length > 0">
      <h2 class="text-lg font-semibold text-gray-800 mb-2">
        Matches Grouped by Document:
      </h2>
      <section class="!space-y-3 mt-4">
        <div
          v-for="(matches, fileName) in results"
          :key="fileName"
          class="mb-5"
        >
          <h3 class="font-bold text-black text-xl mb-1">
            {{ fileName }}
          </h3>
          <ul class="list-disc list-inside ml-6 text-gray-700">
            <li v-for="(match, index) in matches" :key="index">
              <strong>{{ match.keyword }}</strong> on page {{ match.page }}
            </li>
          </ul>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped></style>
