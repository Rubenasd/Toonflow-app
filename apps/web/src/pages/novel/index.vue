<template>
  <main class="novelStudio">
    <header class="studioHeader">
      <div>
        <p class="eyebrow">MANJU AI STUDIO · PHASE 01</p>
        <h1>小说智能改编</h1>
        <p class="subtitle">从原文整理到故事档案、分集规划和剧本初稿。</p>
      </div>
      <div class="headerActions">
        <el-button :disabled="!chapters.length" @click="saveProject">保存项目</el-button>
        <el-button :disabled="!chapters.length" @click="exportProject">导出项目 JSON</el-button>
        <el-button @click="projectInput?.click()">导入项目 JSON</el-button>
        <input ref="projectInput" type="file" accept=".json,application/json" hidden @change="importProject" />
      </div>
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

    <section v-if="chapters.length" class="storyPanel">
      <div class="panelTitle"><div><h2>03 / 故事档案</h2><p>先建立可编辑的改编底稿；人物与关系由创作者确认，避免把文本猜测当成事实。</p></div></div>
      <div class="storyGrid">
        <label>故事一句话梗概<el-input v-model="story.logline" type="textarea" :rows="3" placeholder="主角是谁？想得到什么？最大的阻碍是什么？" /></label>
        <label>核心主题<el-input v-model="story.theme" placeholder="例如：信任、复仇、成长" /></label>
        <label>主线冲突<el-input v-model="story.conflict" type="textarea" :rows="3" placeholder="故事推动主角前进的核心矛盾" /></label>
        <label>世界观 / 故事背景<el-input v-model="story.world" type="textarea" :rows="3" placeholder="时代、地点、规则、特殊设定" /></label>
      </div>
      <div class="panelTitle subTitle"><div><h3>人物档案</h3><p>添加主要人物，并维护身份、目标和人物关系。</p></div><el-button @click="addCharacter">添加人物</el-button></div>
      <div v-if="!story.characters.length" class="emptyHint">尚未添加人物。建议先从主角和主要对手开始。</div>
      <div v-for="person in story.characters" :key="person.id" class="characterRow">
        <el-input v-model="person.name" placeholder="人物姓名" />
        <el-input v-model="person.identity" placeholder="身份 / 角色定位" />
        <el-input v-model="person.goal" placeholder="人物目标 / 动机" />
        <el-input v-model="person.relationship" placeholder="与其他人物的关系" />
        <el-button text type="danger" @click="story.characters = story.characters.filter(item => item.id !== person.id)">删除</el-button>
      </div>
    </section>

    <section v-if="chapters.length" class="episodePanel">
      <div class="panelTitle"><div><h2>04 / 分集策划</h2><p>按章节顺序生成分集草案。每集目标、冲突和结尾钩子均可人工修改。</p></div><el-button type="primary" plain @click="buildEpisodes">重新生成分集草案</el-button></div>
      <div class="episodeToolbar">
        <span>目标集数</span><el-input-number v-model="episodeCount" :min="1" :max="Math.max(1, chapters.length * 3)" />
        <el-text type="info">当前 {{ episodes.length }} 集</el-text>
      </div>
      <div v-if="!episodes.length" class="emptyHint">点击“生成分集草案”，按章节内容创建可编辑的分集结构。</div>
      <article v-for="episode in episodes" :key="episode.id" class="episodeCard">
        <div class="episodeHeading"><strong>第 {{ episode.number }} 集</strong><el-tag effect="plain">{{ episode.chapterRange }}</el-tag></div>
        <label>本集标题<el-input v-model="episode.title" /></label>
        <label>本集目标<el-input v-model="episode.goal" type="textarea" :rows="2" /></label>
        <label>核心冲突<el-input v-model="episode.conflict" type="textarea" :rows="2" /></label>
        <label>结尾钩子<el-input v-model="episode.hook" type="textarea" :rows="2" /></label>
        <div class="episodeSource">原文范围：{{ episode.sourceChapterTitles.join("、") }}</div>
        <el-button text type="primary" @click="createScript(episode)">生成剧本初稿</el-button>
        <div v-if="episode.scenes.length" class="sceneList">
          <div v-for="(scene, index) in episode.scenes" :key="scene.id" class="sceneCard">
            <div class="sceneHeading"><strong>场景 {{ index + 1 }}</strong><el-button text type="danger" @click="episode.scenes.splice(index, 1)">删除场景</el-button></div>
            <el-input v-model="scene.location" placeholder="场景 / 地点" />
            <el-input v-model="scene.action" type="textarea" :rows="3" placeholder="动作与画面描述" />
            <el-input v-model="scene.dialogue" type="textarea" :rows="2" placeholder="对白 / 旁白（可后续补充）" />
          </div>
        </div>
      </article>
    </section>
  </main>
</template>

<script setup lang="ts">
import { ref } from "vue";
import { ElMessage } from "element-plus";

type NovelChapter = { id: string; title: string; content: string };
type Character = { id: string; name: string; identity: string; goal: string; relationship: string };
type Scene = { id: string; location: string; action: string; dialogue: string };
type Episode = { id: string; number: number; title: string; goal: string; conflict: string; hook: string; chapterRange: string; sourceChapterTitles: string[]; scenes: Scene[] };

const fileInput = ref<HTMLInputElement>();
const projectInput = ref<HTMLInputElement>();
const storageKey = "manju-ai-studio:novel-project:v1";
const novelTitle = ref("");
const sourceText = ref("");
const chapters = ref<NovelChapter[]>([]);
const activeChapter = ref(0);
const episodeCount = ref( Math.max(1,  Math.min(12, chapters.value.length || 1)) );
const episodes = ref<Episode[]>([]);
const story = ref({ logline: "", theme: "", conflict: "", world: "", characters: [] as Character[] });

async function importFile(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0];
  if (!file) return;
  sourceText.value = await file.text();
  novelTitle.value = file.name.replace(/\.(txt|md|markdown)$/i, "");
  chapters.value = [];
  episodes.value = [];
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
    if (heading.test(line.trim())) { pushChapter(); title = line.trim().replace(/^#+\s*/, ""); body = []; }
    else body.push(line);
  }
  pushChapter();
  chapters.value = parsed;
  episodeCount.value = Math.max(1, Math.min(12, parsed.length));
  episodes.value = [];
  activeChapter.value = 0;
  ElMessage.success(`已整理 ${parsed.length} 个章节`);
}

function addCharacter() {
  story.value.characters.push({ id: crypto.randomUUID(), name: "", identity: "", goal: "", relationship: "" });
}

function buildEpisodes() {
  if (!chapters.value.length) return;
  const count = Math.max(1, Math.min(episodeCount.value, chapters.value.length * 3));
  const groups: NovelChapter[][] = Array.from({ length: count }, () => []);
  chapters.value.forEach((chapter, index) => groups[Math.min(count - 1, Math.floor(index * count / chapters.value.length))]!.push(chapter));
  episodes.value = groups.filter(group => group.length).map((group, index) => {
    const first = group[0]!;
    const last = group[group.length - 1]!;
    const paragraphs = group.flatMap(chapter => chapter.content.split(/\n+/).map(value => value.trim()).filter(Boolean));
    const opening = paragraphs[0] ?? "";
    const ending = paragraphs[paragraphs.length - 1] ?? "";
    return {
      id: crypto.randomUUID(), number: index + 1,
      title: first.title.replace(/^第.+?[章节回卷部篇集]\s*/, "") || `第 ${index + 1} 集`,
      goal: opening.slice(0, 120) || "请填写本集的剧情目标",
      conflict: paragraphs[Math.min(1, paragraphs.length - 1)]?.slice(0, 160) || "请明确本集的主要矛盾",
      hook: ending.slice(-160) || "请设计推动观众继续观看的结尾悬念",
      chapterRange: group.length === 1 ? first.title : `${first.title} — ${last.title}`,
      sourceChapterTitles: group.map(chapter => chapter.title), scenes: [],
    };
  });
  ElMessage.success(`已生成 ${episodes.value.length} 集分集草案，请逐集审核`);
}

function createScript(episode: Episode) {
  const source = chapters.value.filter(chapter => episode.sourceChapterTitles.includes(chapter.title));
  const paragraphs = source.flatMap(chapter => chapter.content.split(/\n+/).map(value => value.trim()).filter(value => value.length > 12));
  episode.scenes = paragraphs.slice(0, 8).map((paragraph, index) => ({
    id: crypto.randomUUID(),
    location: index === 0 ? "场景待确认" : "地点待确认",
    action: paragraph,
    dialogue: "对白待提取 / 待编写",
  }));
  if (!episode.scenes.length) ElMessage.warning("本集原文内容为空，暂时无法生成场景");
  else ElMessage.success(`已根据原文段落生成 ${episode.scenes.length} 个场景初稿，请补充人物对白和镜头动作`);
}

function projectData() {
  return {
    schemaVersion: 2, title: novelTitle.value.trim() || "未命名小说",
    exportedAt: new Date().toISOString(), sourceCharacterCount: sourceText.value.length,
    sourceText: sourceText.value, story: story.value, chapters: chapters.value, episodes: episodes.value,
  };
}

function saveProject() {
  try {
    localStorage.setItem(storageKey, JSON.stringify(projectData()));
    ElMessage.success("项目已保存到当前浏览器");
  } catch (error) {
    ElMessage.error(error instanceof Error && error.name === "QuotaExceededError" ? "浏览器本地空间不足，请导出 JSON 备份" : "保存失败，请导出 JSON 备份");
  }
}

async function importProject(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files?.[0];
  if (!file) return;
  try {
    const data = JSON.parse(await file.text()) as Record<string, unknown>;
    if (data.schemaVersion !== 1 && data.schemaVersion !== 2) throw new Error("不支持的项目版本");
    if (!Array.isArray(data.chapters) || !data.chapters.every(item => item && typeof item.title === "string" && typeof item.content === "string")) {
      throw new Error("项目文件缺少有效章节数据");
    }
    novelTitle.value = typeof data.title === "string" ? data.title : "导入的漫剧项目";
    sourceText.value = typeof data.sourceText === "string" ? data.sourceText : data.chapters.map(item => item.content).join("\\n\\n");
    chapters.value = data.chapters as NovelChapter[];
    story.value = data.story && typeof data.story === "object"
      ? { logline: "", theme: "", conflict: "", world: "", characters: [], ...(data.story as typeof story.value) }
      : { logline: "", theme: "", conflict: "", world: "", characters: [] };
    episodes.value = Array.isArray(data.episodes) ? data.episodes as Episode[] : [];
    episodeCount.value = Math.max(1, episodes.value.length || Math.min(12, chapters.value.length));
    activeChapter.value = 0;
    ElMessage.success("项目导入成功");
  } catch (error) {
    ElMessage.error(error instanceof Error ? error.message : "项目文件读取失败");
  } finally {
    input.value = "";
  }
}

function exportProject() {
  const data = projectData();
  const url = URL.createObjectURL(new Blob([JSON.stringify(data, null, 2)], { type: "application/json" }));
  const anchor = document.createElement("a");
  anchor.href = url;
  anchor.download = `${data.title.replace(/[\\/:*?"<>|]/g, "_")}-漫剧改编项目.json`;
  anchor.click();
  URL.revokeObjectURL(url);
}
</script>

<style lang="scss" scoped>
.novelStudio { max-width: 1180px; margin: 0 auto; padding: 36px; display: grid; gap: 22px; color: var(--el-text-color-primary); }
.studioHeader, .panelTitle, .importFooter, .episodeToolbar, .episodeHeading, .sceneHeading { display: flex; align-items: center; justify-content: space-between; gap: 18px; }
.headerActions { display: flex; flex-wrap: wrap; justify-content: flex-end; gap: 8px; }
.studioHeader h1 { margin: 5px 0; font-size: 30px; }
.eyebrow { margin: 0; color: var(--el-color-primary); font-size: 11px; letter-spacing: 2px; }
.subtitle, .panelTitle p { margin: 5px 0 0; color: var(--el-text-color-secondary); font-size: 13px; }
.importPanel, .chapterPanel, .storyPanel, .episodePanel { padding: 22px; border: 1px solid var(--el-border-color); border-radius: 12px; background: var(--el-bg-color-overlay); }
.panelTitle { margin-bottom: 18px; }
.panelTitle h2 { margin: 0; font-size: 16px; }
.panelTitle h3 { margin: 0; font-size: 15px; }
.importPanel > .el-input { margin-bottom: 12px; }
.importFooter { justify-content: flex-end; color: var(--el-text-color-secondary); font-size: 12px; }
.chapterLayout { display: grid; grid-template-columns: 250px minmax(0, 1fr); gap: 18px; }
.chapterList { display: flex; flex-direction: column; gap: 6px; max-height: 620px; overflow: auto; }
.chapterList button { padding: 12px; display: grid; gap: 5px; text-align: left; color: inherit; border: 1px solid var(--el-border-color-lighter); border-radius: 8px; background: transparent; cursor: pointer; }
.chapterList button.active { border-color: var(--el-color-primary); background: var(--el-color-primary-light-9); }
.chapterList span, .chapterList small, .episodeSource { color: var(--el-text-color-secondary); font-size: 11px; }
.chapterList strong { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font-size: 13px; }
.chapterEditor { display: grid; align-content: start; gap: 12px; min-width: 0; }
.storyGrid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; }
.storyGrid label, .episodeCard label { display: grid; gap: 7px; font-size: 13px; font-weight: 600; }
.subTitle { margin-top: 26px; }
.emptyHint { padding: 22px; color: var(--el-text-color-secondary); text-align: center; background: var(--el-fill-color-lighter); border-radius: 8px; }
.characterRow { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)) auto; gap: 8px; margin: 10px 0; }
.episodeToolbar { justify-content: flex-start; margin-bottom: 16px; }
.episodeCard { display: grid; gap: 12px; padding: 18px; margin-top: 12px; border: 1px solid var(--el-border-color); border-radius: 10px; }
.episodeHeading { justify-content: flex-start; }
.episodeSource { padding: 8px 0; }
.sceneList { display: grid; gap: 10px; padding-top: 10px; border-top: 1px solid var(--el-border-color); }
.sceneCard { display: grid; gap: 8px; padding: 12px; border-radius: 8px; background: var(--el-fill-color-lighter); }
@media (max-width: 760px) {
  .novelStudio { padding: 18px; }
  .studioHeader, .panelTitle { align-items: flex-start; flex-wrap: wrap; }
  .headerActions { justify-content: flex-start; }
  .chapterLayout, .storyGrid { grid-template-columns: 1fr; }
  .chapterList { max-height: 220px; }
  .characterRow { grid-template-columns: 1fr 1fr; }
}
</style>
