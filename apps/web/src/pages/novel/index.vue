<template>
  <main class="novelStudio">
    <header class="studioHeader">
      <div>
        <p class="eyebrow">MANJU AI STUDIO · PHASE 01</p>
        <h1>小说智能改编</h1>
        <p class="subtitle">导入原文，识别章节并整理成可编辑的故事素材。</p>
      </div>
      <el-button :disabled="!chapters.length" @click="exportProject">导出改编素材 JSON</el-button>
    </header>

    <section class="importPanel">
      <div class="panelTitle">
        <div><h2>01 / 导入小说</h2><p>支持 TXT、Markdown。原文只在当前浏览器处理中，不会自动上传。</p></div>
        <el-button type="primary" @click="fileInput?.click()">选择文件</el-button>
        <input ref="fileInput" type="file" accept=".txt,.md,.markdown,text/plain,text/markdown" hidden @change="importFile" />
      </div>
      <el-input v-model="novelTitle" placeholder="作品名称" />
      <el-input v-model="sourceText" type="textarea" :autosize="{ minRows: 8, maxRows: 18 }" resize="vertical" placeholder="也可以直接粘贴小说正文……" />
      <div class="importFooter">
        <span>{{ sourceText.length.toLocaleString() }} 字符 · {{ chapters.length }} 个章节</span>
        <el-button type="primary" plain :disabled="!sourceText.trim()" @click="parseChapters">识别章节</el-button>
      </div>
    </section>

    <section v-if="chapters.length" class="chapterPanel">
      <div class="panelTitle"><div><h2>02 / 章节整理</h2><p>可修改章节标题和正文；未识别到章节标题时，会将全文作为一章。</p></div></div>
      <div class="chapterLayout">
        <nav class="chapterList" aria-label="章节列表">
          <button v-for="(chapter, index) in chapters" :key="chapter.id" type="button" :class="{ active: activeChapter === index }" @click="activeChapter = index">
            <span>第 {{ index + 1 }} 章</span><strong>{{ chapter.title }}</strong><small>{{ chapter.content.length.toLocaleString() }} 字符</small>
          </button>
        </nav>
        <div class="chapterEditor">
          <el-input v-model="chapters[activeChapter]!.title" aria-label="章节标题" />
          <el-input v-model="chapters[activeChapter]!.content" type="textarea" :autosize="{ minRows: 16, maxRows: 30 }" resize="vertical" aria-label="章节正文" />
        </div>
      </div>
    </section>

    <section v-if="chapters.length" class="nextStep">
      <div><span class="stepNumber">03</span><div><strong>下一步：故事结构化</strong><p>章节整理完成后，可导出素材交给后续故事分析流程。</p></div></div>
      <el-button type="primary" @click="exportProject">导出当前素材</el-button>
    </section>
  </main>
</template>

<script setup lang="ts">
import { ref } from "vue";
import { ElMessage } from "element-plus";

type NovelChapter = { id: string; title: string; content: string };

const fileInput = ref<HTMLInputElement>();
const novelTitle = ref("");
const sourceText = ref("");
const chapters = ref<NovelChapter[]>([]);
const activeChapter = ref(0);

async function importFile(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (!file) return;
  sourceText.value = await file.text();
  novelTitle.value = file.name.replace(/\.(txt|md|markdown)$/i, "");
  chapters.value = [];
  activeChapter.value = 0;
  (event.target as HTMLInputElement).value = "";
}

function parseChapters() {
  const lines = sourceText.value.replace(/\r\n?/g, "\n").split("\n");
  const heading = /^(?:第[零〇一二三四五六七八九十百千万\d]+[章节回卷部篇集].*|chapter\s+\d+.*|#+\s+.+)$/i;
  const parsed: NovelChapter[] = [];
  let title = "";
  let body: string[] = [];

  function pushChapter() {
    const text = body.join("\n").trim();
    if (title || text) parsed.push({ id: crypto.randomUUID(), title: title || `章节 ${parsed.length + 1}`, content: text });
  }

  for (const line of lines) {
    if (heading.test(line.trim())) {
      pushChapter();
      title = line.trim().replace(/^#+\s*/, "");
      body = [];
    } else {
      body.push(line);
    }
  }
  pushChapter();
  chapters.value = parsed;
  activeChapter.value = 0;
  ElMessage.success(`已整理 ${parsed.length} 个章节`);
}

function exportProject() {
  const data = {
    schemaVersion: 1,
    title: novelTitle.value.trim() || "未命名小说",
    exportedAt: new Date().toISOString(),
    sourceCharacterCount: sourceText.value.length,
    chapters: chapters.value,
  };
  const url = URL.createObjectURL(new Blob([JSON.stringify(data, null, 2)], { type: "application/json" }));
  const anchor = document.createElement("a");
  anchor.href = url;
  anchor.download = `${data.title.replace(/[\\/:*?"<>|]/g, "_")}-改编素材.json`;
  anchor.click();
  URL.revokeObjectURL(url);
}
</script>

<style lang="scss" scoped>
.novelStudio { max-width: 1180px; margin: 0 auto; padding: 36px; display: grid; gap: 22px; color: var(--el-text-color-primary); }
.studioHeader, .panelTitle, .importFooter, .nextStep, .nextStep > div { display: flex; align-items: center; justify-content: space-between; gap: 18px; }
.studioHeader h1 { margin: 5px 0; font-size: 30px; }
.eyebrow { margin: 0; color: var(--el-color-primary); font-size: 11px; letter-spacing: 2px; }
.subtitle, .panelTitle p, .nextStep p { margin: 5px 0 0; color: var(--el-text-color-secondary); font-size: 13px; }
.importPanel, .chapterPanel, .nextStep { padding: 22px; border: 1px solid var(--el-border-color); border-radius: 12px; background: var(--el-bg-color-overlay); }
.panelTitle { margin-bottom: 18px; }
.panelTitle h2 { margin: 0; font-size: 16px; }
.importPanel > .el-input { margin-bottom: 12px; }
.importFooter { justify-content: flex-end; color: var(--el-text-color-secondary); font-size: 12px; }
.chapterLayout { display: grid; grid-template-columns: 250px minmax(0, 1fr); gap: 18px; }
.chapterList { display: flex; flex-direction: column; gap: 6px; max-height: 620px; overflow: auto; }
.chapterList button { padding: 12px; display: grid; gap: 5px; text-align: left; color: inherit; border: 1px solid var(--el-border-color-lighter); border-radius: 8px; background: transparent; cursor: pointer; }
.chapterList button.active { border-color: var(--el-color-primary); background: var(--el-color-primary-light-9); }
.chapterList span, .chapterList small { color: var(--el-text-color-secondary); font-size: 11px; }
.chapterList strong { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font-size: 13px; }
.chapterEditor { display: grid; align-content: start; gap: 12px; min-width: 0; }
.nextStep { background: var(--el-fill-color-lighter); }
.nextStep > div { justify-content: flex-start; }
.stepNumber { color: var(--el-color-primary); font-size: 24px; font-weight: 700; }
@media (max-width: 760px) {
  .novelStudio { padding: 18px; }
  .studioHeader, .panelTitle { align-items: flex-start; flex-wrap: wrap; }
  .chapterLayout { grid-template-columns: 1fr; }
  .chapterList { max-height: 220px; }
  .nextStep { align-items: flex-start; flex-direction: column; }
}
</style>
