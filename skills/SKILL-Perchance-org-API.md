---
name: perchance-api
description: >
  Use this skill whenever working with Perchance.org — building generators, AI chat apps, plugins,
  or any JavaScript code that runs on perchance.org. Covers the Perchance DSL syntax, the ai-text-plugin
  API, text-to-image plugin, character/thread/message data models, IndexedDB (Dexie.js) patterns,
  hierarchical summarization, lorebooks, memory systems, streaming responses, sandboxed custom code,
  and share-link/upload patterns. Trigger this skill for any mention of perchance, ai-text-plugin,
  perchance generator, ai-character-chat, or building a Perchance-hosted AI app. Also trigger when
  the user shows code that uses `root.aiTextPlugin`, `root.textToImagePlugin`, Perchance list syntax,
  or the `{import:...}` pattern.
---

# Perchance API & AI Chat Skill

This skill encodes the complete architecture of a production Perchance AI chat application, distilled
from real source code. Use it to build, debug, or extend any Perchance-hosted generator or AI app.

---

## 1 · Perchance DSL Fundamentals

Perchance pages have two zones: the **top generator** (Perchance DSL) and the **HTML panel** (standard HTML/JS).

### 1.1 List Syntax (top editor)
```
listName
  item one
  item two
  {nestedList}         // embed another list by name
  {import:plugin-name} // import a plugin

// Calling a function defined in a list:
result = root.myFunction(arg1, arg2)
```

### 1.2 Async Functions in Lists
```
async myFunction(opts) =>
  if(!opts) opts = {};
  let result = await someAsyncThing();
  return result;
```
- **All lines are auto-dedented** inside list functions — if you move code to a `<script>`, re-add indentation manually.
- Access sibling list items/functions via `root.listName` or `root.functionName()`.

### 1.3 Key Built-in Plugins (imported at top)
```
loadDependencies = {import:ai-character-chat-dependencies-v1} // bundles Dexie.js, DOMPurify, etc.
aiTextPlugin = {import:ai-text-plugin}
textToImagePlugin = {import:text-to-image-plugin}
uploadPlugin = {import:upload-plugin}          // for character share links
superFetch = {import:super-fetch-plugin}        // bypass CORS
commentsPlugin = {import:comments-plugin}
tabbedCommentsPlugin = {import:tabbed-comments-plugin-v1}
fullscreenButtonPlugin = {import:fullscreen-button-plugin}
combineEmojis = {import:combine-emojis-plugin}
bugReport = {import:bug-report-plugin}          // browser debug info for bug reports
dynamicImport = {import:dynamic-import-plugin}  // lazy-load other generators at runtime
```

### 1.4 `dynamicImport` — Lazy Loading External Generators
Use `dynamicImport` to load another Perchance generator's lists at runtime, avoiding page-load cost:
```
customBots
  FreemanBots = [dynamicImport('sjjhkyohfs')]
  BegginnerBots = [dynamicImport('7876aw570s')]
```
Equivalent static import (loads at page start, not lazy):
```
// FreemanBots = {import:sjjhkyohfs}
```
Use `dynamicImport` for optional/large dependencies; use `{import:...}` for required dependencies.

### 1.5 Meta / SEO
```
$meta
  header
    mode = minimal
  async dynamic(inputs) =>      // MUST be fully self-contained — no outside refs
    let urlParams = inputs.urlParams;
    return { title: "...", description: "...", image: "..." };
```

**Critical `$meta.dynamic` constraint:** This function cannot reference `root.*`, other lists, or
any global variables defined outside itself. If your generator uses a `urlNamedCharacters` list,
you must **duplicate the mapping inline** inside `$meta.dynamic` — it cannot read the list:
```
// Top list:
urlNamedCharacters
  ai-adventure = 6c2f68e41de41e75a51971487c97b2d9.gz
  therapist = 5cdaa39f9aabc7424c3b2e1b780a1e29.gz
  // NOTE: must add named chars to $meta.dynamic too

// $meta.dynamic — must re-declare the same map:
async dynamic(inputs) =>
  let urlNamedCharacters = {         // can't use root.* — duplicate it here
    "ai-adventure": "6c2f68e41de41e75a51971487c97b2d9.gz",
    "therapist": "5cdaa39f9aabc7424c3b2e1b780a1e29.gz",
  };
  let fileName = urlNamedCharacters[inputs.urlParams.char];
  ...
```

---

## 2 · `ai-text-plugin` API  ← Core AI generation

```js
// Basic call (non-streaming):
let data = await root.aiTextPlugin({
  instruction: "System context + task description",
  startWith: "Text the AI will continue from",
  stopSequences: ["\n\n[[", "\n[["],
  hideStartWith: true,   // don't echo startWith back in generatedText
});
let text = data.generatedText;    // string
let stopReason = data.stopReason; // "stop_sequence" | "max_tokens" | "error"

// Streaming call:
let streamObj = root.aiTextPlugin({
  instruction,
  startWith,
  stopSequences,
  hideStartWith: true,
  onChunk: (data) => {
    if(data.isFromStartWith) return;   // skip echo of startWith
    let chunk = data.textChunk;        // string delta
    // update UI here
  },
});
streamObj.stop(); // call to abort early
let finalData = await streamObj;      // resolves when complete

// Preload (warm up inference engine):
// On mobile, delay preload to avoid slowing page load:
if(window.innerWidth < 500) setTimeout(() => root.aiTextPlugin({preload:true}), 5000);
else root.aiTextPlugin({preload:true});

// Token utilities:
const { countTokens, idealMaxContextTokens } = root.aiTextPlugin({ getMetaObject: true });
// countTokens(text: string) => number
// idealMaxContextTokens => ~4000–8000

// Text embedding for semantic search:
// CORRECT signature — pass an object, not a bare array:
let [vector] = await window.embedTexts({ textArr: ["text a"], modelName: thread.textEmbeddingModelName });
// returns Float32Array[]
let distance = cosineDistance(vec1, vec2); // lower = more similar

// Always guard before embedding — not all browsers can load the model:
if(window.textEmbedderFunction) {
  let [embedding] = await window.embedTexts({ textArr: [memText], modelName: thread.textEmbeddingModelName });
}
```

### 2.1 Instruction Patterns (used in production)

**Chat completion** — the canonical `[[CharacterName]]: message` format:
```js
let instruction = `
<MESSAGES>
[[User]]: Hello!
[[Chloe]]: Hi there, how can I help?
</MESSAGES>

REMINDER for writing Chloe's messages: "Keep replies short and in character."

>>> TASK: Your task is to write the next 3 messages in this chat.
`.trim();

let startWith = `[[Chloe]]:`;   // AI continues from here
```

**Summarisation task** — the `[A]/[B]/[C]` block-labeling format (from production):
```js
// startWith contains real [A] and [B] examples so the AI can see the pattern:
let startWith = `
>>> FULL TEXT of [A]: <previously seen text>
>>> SUMMARY of [A]: <already written summary>
---
>>> FULL TEXT of [B]: <another block of text>
>>> SUMMARY of [B]: <already written summary>
---
>>> FULL TEXT of [C]: ${messagesToSummarize}
>>> SUMMARY of [C]:`.trim();

// Append to force quality (in production: slice off last char and append this):
startWith = startWith + " (full, natural, readable sentences with correct grammar):";

let stopSequences = ["\n\n", "\n---", "\n>>> FULL TEXT", "FULL TEXT"];

// Post-process: strip artifacts from returned summary text
let summary = data.generatedText.trim()
  .replace(/\n+/g, " ").trim()
  .replace(/---$/, "")
  .replace(">>> FULL TEXT", "").replace("FULL TEXT", "").trim()
  .replaceAll(/ *[—–] */g, ", ").trim();
```

**Summarisation instruction tips** (improves quality when included):
```
TIP: In order to ensure the summary is half the length of the full text, consider what's happening
at a "higher level", rather than simply re-stating individual facts in terse form.
TIP: Ensure summaries use full, natural, readable sentences with correct grammar.
NOTE: Don't append commentary or word counts. Just do the task and end your response.
IMPORTANT: Avoid repetition within summaries! Remove or ignore erroneously repeated elements.
```

**Auto-fix repetition in summaries:**
```js
// Detect: if final 30 chars appear many times earlier in text, repetition is likely
if(summary.split(summary.slice(-30)).length > 5) {
  let result = await root.aiTextPlugin({
    instruction: [
      `Does the following story summary snippet shown within <story_summary_snippet>...</story_summary_snippet>`,
      `include erroneous/unnecessary repetition? If so, respond with fixed text within`,
      `<fixed_story_summary_snippet>...</fixed_story_summary_snippet>. If fine, respond with exactly 'no_repetition'.`,
      ``,
      `<story_summary_snippet>`,
      summary,
      `</story_summary_snippet>`,
    ].join("\n"),
    stopSequences: ["</fixed_story_summary_snippet>"],
  });
  let fixedSummary = result.generatedText
    .match(/<fixed_story_summary_snippet>(.+)<\/fixed_story_summary_snippet>/s)?.[1].trim();
  if(fixedSummary) summary = fixedSummary;
}
```

**Memory / lore extraction:**
```js
let instruction = `
@@@ TASK: Condense the *NEW_TEXT* below into up to 3 lore/memory/fact entries for a facts database.
- Only extract "timeless facts" — things that will ALWAYS be true.
- Example timeless: "Bob was born in Paris". Example NOT timeless: "Bob is hungry right now".
- Each entry must be fully self-contained (may be read in isolation — include full context).
- Use actual names, not pronouns. "Bob said to Alice..." not "He said to her..."
- Each entry: no more than 3 sentences.
- If unsure if a fact is timeless, add context: "Bob was kind to Alice when he first met her."
- IMPORTANT: Do NOT repeat things already known from above context. Only extract NEW information.
# NEW_TEXT:
${messagesSummarizedText}
Use this format:
# Lore/memory entries from NEW_TEXT:
1. <something we can deduce directly from NEW_TEXT>
2. <something else>
3. <another thing>
`.trim();

let startWith = `# Lore/memory entries from NEW_TEXT:\n1.`;
let stopSequences = ["\n4."];

// Post-process extracted memories:
memories = ("1."+data.generatedText).trim()
  .split("\n").map(l => l.trim())
  .filter(l => l && /^[0-9]\. .+/.test(l))
  .map(l => l.replace(/^[0-9]\. /, "").replaceAll(/ *[—–] */g, ", "));
```

**Shared prefix cache** — structure summary AND memory calls to share a common prefix.
This avoids re-tokenising the shared context for each call:
```js
let sharedContextPrefixText = [
  `# Extra context:\n${extraContext}`,
  `# Prior events summary:`,
  (priorSummaries.length > 0 ? priorSummaries : ["(None.)"]).join("\n"),
].join("\n");

// Summary call uses: [sharedContextPrefixText, "", summaryTaskPrompt]
// Memory call uses:  [sharedContextPrefixText, "", memoryTaskPrompt]
// Both share the same prefix → better prefix-cache hit rates
```

---

## 3 · `text-to-image-plugin` API

```js
// In messages, AI writes image tags — the plugin renders them automatically:
// <image>A detailed description of the scene</image>

// To trigger image generation directly:
let imageUrl = await root.textToImagePlugin({
  prompt: "anime girl in a sunny field",
  // negative: "blurry, bad quality",
  // style: "photorealistic",
});
```

Character-level image controls (set on character object):
- `imagePromptPrefix` — prepended to every image prompt
- `imagePromptSuffix` — appended to every image prompt
- `imagePromptTriggers` — `"keyword: description\nother: @prepend desc"` format

---

## 4 · Data Model (IndexedDB via Dexie.js)

```js
// DB opened at app startup:
const db = new Dexie("chatbot-ui-v1");
db.version(N).stores({
  characters: "++id, name, uuid",
  threads: "++id, characterId, lastViewTime, lastMessageTime",
  messages: "++id, threadId, characterId, order",
  memories: "++id, threadId",
  lore: "++id, bookId, bookUrl",
  summaries: "hash",
  usageStats: "++id, threadId",
  misc: "key",        // general key-value store for app-level settings
});
```

**`db.misc` usage** — key-value store for app-level settings and custom code:
```js
// Read:
let value = (await db.misc.get("customPostPageLoadMainThreadCode"))?.value || "";
// Write:
await db.misc.put({ key: "customPostPageLoadMainThreadCode", value: "..." });
// Run custom post-load code if set (allows user-injected JS at startup):
if(customPostPageLoadMainThreadCode.trim()) eval(customPostPageLoadMainThreadCode);
```

### 4.1 Character Object Schema
```js
{
  id, uuid,
  name: "Chloe",
  roleInstruction: "Character description / system prompt (keep < 1000 words)",
  reminderMessage: "Short reminder note appended near the end (< 100 words)",
  generalWritingInstructions: "@roleplay1" | "@roleplay2" | "custom text",
  initialMessages: [{ author:"user"|"ai"|"system", content:"..." }],
  avatar: { url: "https://...", size: 1, shape: "square" },
  userCharacter: { name, roleInstruction, reminderMessage, avatar: { url } },
  systemCharacter: { avatar: {} },
  modelName: "good" | "great",  // maps to underlying LLM
  scene: { background: { url }, music: { url } },
  loreBookUrls: ["https://user.uploads.dev/file/xxx.txt"],
  autoGenerateMemories: "none" | "enabled",
  textEmbeddingModelName: "default",
  maxParagraphCountPerMessage: null | 1 | 2 | 3 | 4,
  streamingResponse: true,
  customCode: "",   // JavaScript run in sandboxed iframe
  // Default custom code — apply named global preset if character has no custom code:
  // character.customCode = character.customCode?.length
  //   ? character.customCode
  //   : window.globalCustomJavaScripts["@removeHyphens"];
  imagePromptPrefix: "",
  imagePromptSuffix: "",
  imagePromptTriggers: "",
  metaTitle: "",        // optional — overrides SEO title on this character's share URL
  metaDescription: "", // optional — overrides SEO description
  metaImage: "",        // optional — overrides SEO image
  customData: {},
  folderPath: "",
  creationTime: Date.now(),
  lastMessageTime: Date.now(),
}
```

### 4.2 Message Object Schema
```js
{
  id, threadId, characterId,
  message: "Text of the message",
  name: null,        // name override (null = use character default)
  order: id,         // used for sorting; can be non-contiguous
  hiddenFrom: [],    // [] | ["ai"] | ["user"]
  expectsReply: undefined | true | false,
  variants: [null],  // swipe variants; null = current message
  summariesEndingHere: {},   // { level: "summary text" }
  memoriesEndingHere: {},    // { level: [{text, embedding}] } — parallel to summariesEndingHere
  memoryIdBatchesUsed: [],
  loreIdsUsed: [],
  memoryQueriesUsed: [],
  messageIdsUsed: [],
  scene: null,
  avatar: {},
  customData: {},
  wrapperStyle: "",
  instruction: null,  // replyInstruction used when message was generated
}
```

**`memoriesEndingHere`** — memories extracted from the same block that was summarized:
```js
// During summarization, push memories to the last-summarized message:
if(!m.memoriesEndingHere) m.memoriesEndingHere = {};
if(!m.memoriesEndingHere[level]) m.memoriesEndingHere[level] = [];
m.memoriesEndingHere[level].push(
  ...memories.map(memText => ({ text: memText, embedding: null }))
  // embedding: null here — computed lazily only at DB write time
);

// At DB write time, compute embeddings (only if browser supports it):
if(window.textEmbedderFunction && m.memoriesEndingHere) {
  for(let lvl in m.memoriesEndingHere) {
    for(let memory of m.memoriesEndingHere[lvl]) {
      if(!memory.embedding) {
        let [embedding] = await window.embedTexts({
          textArr: [memory.text],
          modelName: thread.textEmbeddingModelName,
        });
        memory.embedding = embedding;
      }
    }
  }
}
await db.messages.update(m.id, {
  summariesEndingHere: m.summariesEndingHere,
  memoriesEndingHere: m.memoriesEndingHere,
});
```

### 4.3 Thread Object Schema
```js
{
  id, characterId,
  name: "Thread name",
  modelName, textEmbeddingModelName,
  character: { /* thread-level character overrides */ },
  userCharacter: { name, roleInstruction, reminderMessage, avatar: {} },
  systemCharacter: { avatar: {} },
  isFav: false,
  folderPath: "",
  lastViewTime, lastMessageTime,
  currentSummaryHashChain: [],
  customCodeWindow: { visible: false, width: null },
  customData: {},
}
```

---

## 5 · Message Format (wire format sent to AI)

Messages are serialised as plain text before being sent to `aiTextPlugin`:
```
[[CharacterName]]: Message content here.

[[AnotherCharacter]]: Their reply.

[[CharacterName]]: Next line...
```

- Separator: `\n\n` between messages
- Stop sequences: `["\n\n[[", "\n[["]` (add `"\n\n"` to limit to 1 paragraph)
- `hiddenFrom: ["ai"]` messages are filtered out before sending
- `<!--hidden-from-ai-start-->…<!--hidden-from-ai-end-->` strips inline sections
- Template variables: `{{user}}` → userName, `{{char}}` → characterName

---

## 6 · Hierarchical Summarisation

Long conversations are compressed in background after every reply.

### 6.1 Concept
```
Level 0 = raw messages (bottom of stack)
Level 1 = summaries of ~1500-char blocks of raw messages
Level 2 = summaries of level-1 summaries
...
```

### 6.2 Global State Object (per thread)
```js
if(!window.__aiHierarchicalSummaryStuff) window.__aiHierarchicalSummaryStuff = {};
if(!window.__aiHierarchicalSummaryStuff[threadId]) {
  window.__aiHierarchicalSummaryStuff[threadId] = {
    summariesReadyToInject: [],   // pending — batched before DB write
    alreadyDoingSummary: false,   // mutex — only one bg summary at a time
  };
}
```

### 6.3 When to Summarize
```js
const { countTokens, idealMaxContextTokens } = root.aiTextPlugin({ getMetaObject: true });
// Leave an 800-token buffer to prevent prefix-cache invalidation on every message:
let tokenBudget = idealMaxContextTokens - 800;

let currentLength = countTokens(
  messageTextWithSummaryReplacements.join("\n\n") + (opts.extraTextForAccurateTokenCount || "")
);
if(currentLength < tokenBudget) {
  return; // no summarization needed
}
// Trigger background summarization (in an IIFE so it doesn't block reply generation)
```

### 6.4 Batch Injection (protects prefix cache)
```js
// Only write summaries to DB when >= 3 are ready.
// Avoids invalidating the prefix cache after every single message.
if(window.__aiHierarchicalSummaryStuff[threadId].summariesReadyToInject.length >= 3) {
  for(let m of messagesToUpdate) {
    await db.messages.update(m.id, { summariesEndingHere: m.summariesEndingHere });
  }
  window.__aiHierarchicalSummaryStuff[threadId].summariesReadyToInject = [];
}
```

### 6.5 Block Size Heuristic
```js
const numCharsToSummarizeAtATime = 1500;
// Don't go higher — the summary call's own context can overflow when hierarchy is deep.

// Trim final block to target size:
while(block.length > 1) {
  let numChars = block.reduce((a, v) => a + v.length, 0);
  if(numChars < numCharsToSummarizeAtATime) break;
  // Drop half at once if way too big (speed optimization):
  if(numChars > numCharsToSummarizeAtATime * 10) {
    let half = Math.floor(block.length / 2);
    for(let j = 0; j < half; j++) block.pop();
  } else {
    block.pop();
  }
}
```

### 6.6 `getMessageObjsWithoutSummarizedOnes` — Context Reconstruction Algorithm
```js
getMessageObjsWithoutSummarizedOnes(messages, opts) =>
  // Returns minimal set of messages/summaries needed to reconstruct context.
  // Walks BACKWARDS, collecting messages while monotonically climbing levels.
  messages = messages.slice(0);
  let result = [];
  let highestLevelSeen = 0;

  while(messages.length > 0) {
    let m = messages.pop();
    let level = m.summariesEndingHere
      ? Math.max(...Object.keys(m.summariesEndingHere).map(Number))
      : 0;
    if(level < (opts?.minimumMessageLevel || 0)) continue;
    if(level >= highestLevelSeen) {
      result.unshift(m);
      highestLevelSeen = level;
    }
    // When we hit a higher-level summary, it "covers" all previous raw messages.
    // By monotonically climbing, we naturally prefer summaries over raw messages.
  }
  return result;
  // NOTE: When using result for inference, read the HIGHEST level summary from each message:
  // let text = (level > 0) ? m.summariesEndingHere[level] : m.content;
```

### 6.7 Sanity Check Before Injecting Summaries
```js
// Verify that the last summarized message still matches. If the user edited messages,
// discard the summary and recompute it next time.
let expected = level === 1
  ? `${lastMessageObj.name}: ${lastMessageObj.content}`
  : lastMessageObj.summariesEndingHere[level - 1];

if(expected.trim() === lastSummarizedText.trim()) {
  // safe to inject
} else {
  console.warn("Content mismatch — summary discarded. Will recompute next send.");
}
```

---

## 7 · Memory & Lore

**Associative memory** — facts extracted from past conversations, stored in `db.memories`,
retrieved via text embedding similarity search before each reply:
```js
let memories = await db.memories.where("threadId").equals(threadId).toArray();
// Each memory: { id, threadId, text: "Bob was born in Paris" }
// Retrieve relevant subset: embed query, sort by cosineDistance, take top N
```

**Lorebooks** — static fact files hosted at `user.uploads.dev`, loaded and embedded
at thread start:
```js
// Each line is a fact; blank lines separate entries.
// Loaded from character.loreBookUrls[], cached in db.lore
```

**Injection into prompt** (when relevant items found):
```
<ignore_this_if_irrelevant>
[MEMORIES & LORE]
• Bob was born in Paris (memory)
• The castle has three towers (lore)
</ignore_this_if_irrelevant>
```

---

## 8 · File Hosting & Share Links

```js
// Upload a Blob to Perchance CDN:
let { url, size, error } = await uploadPlugin(blob);
// url format: "https://user.uploads.dev/file/<filename>"

// Handle upload errors (including content moderation):
if(error) {
  alert(`Error: ${error}${error === "disallowed_content"
    ? ". If you believe this is incorrect, you may need to edit the character description to"
      + " explicitly state that the character is 18 or older, since the moderation system can"
      + " make mistakes if there is ambiguity."
    : ""}`);
  return;
}

// Share link format:
// https://perchance.org/<generator-name>?data=<CharName>~<filename>.gz

// Full share link generation:
async function generateShareLink(json) {
  if(!window.CompressionStream) {
    alert("Share links require a modern browser. Please upgrade or switch from Safari to Chrome.");
    return;
  }
  let loadingModal = createLoadingModal("⏳ Generating share link...");

  // Also build a hash-based URL (for debugging):
  let urlHashData = encodeURIComponent(JSON.stringify(json))
    .replace(/[!'()*]/g, c => '%' + c.charCodeAt(0).toString(16));
  console.log("shareUrl (hash version):", `https://perchance.org/${window.generatorName}#${urlHashData}`);

  // Convert to blob, compress, upload:
  let jsonString = JSON.stringify(json);
  let blob = await fetch("data:text/plain;charset=utf-8," + jsonString.replace(/#/g, "%23"))
    .then(r => r.blob());
  let compressed = await compressBlobWithGzip(blob);
  let { url, error } = await uploadPlugin(compressed);
  loadingModal.delete();
  if(error) { /* handle error */ return; }

  let fileName = url.replace("https://user.uploads.dev/file/", "");
  // Character name in URL is cosmetic only — sanitize but doesn't affect data:
  let charName = json.addCharacter.name.replace(/\s+/g, "_").replaceAll("~", "");
  return `https://perchance.org/${window.generatorName}?data=${charName}~${fileName}`;
}

// Loading from a share link URL:
async function loadDataFromShareUrl() {
  let searchParams = new URL(window.location.href).searchParams;
  let dataParam = searchParams.get("data");
  if(!dataParam && searchParams.get("char")) {
    // Named characters: ?char=ai-adventure → look up gz filename from urlNamedCharacters list
    dataParam = "foo~" + urlNamedCharacters[searchParams.get("char")];
  }
  let fileName = dataParam.split("~").slice(-1)[0];
  let fileUrl = "https://user.uploads.dev/file/" + fileName;

  let blob = await fetch(fileUrl, {
    signal: AbortSignal.timeout ? AbortSignal.timeout(15000) : null
  }).then(r => r.ok ? r.blob() : null).catch(console.error);

  if(!blob) {
    await confirmAsync(
      "File not found. Check if it has been quarantined:\nperchance.org/quarantined-files",
      { hideCancel: true }
    );
    return null;
  }
  let decompressed = await decompressBlobWithGzip(blob);
  return JSON.parse(await decompressed.text());
}

// Compression helpers:
async function compressBlobWithGzip(blob) {
  const cs = new CompressionStream("gzip");
  let out = await new Response(blob.stream().pipeThrough(cs)).blob();
  return new Blob([out], { type: "application/gzip" });
}
async function decompressBlobWithGzip(blob) {
  const ds = new DecompressionStream("gzip");
  return await new Response(blob.stream().pipeThrough(ds)).blob();
}
```

---

## 9 · Sandboxed Custom Code

Characters can run user-supplied JavaScript in a hidden sandboxed iframe:
```js
// Evaluation:
let result = await root.evaluatePerchanceTextInSandbox(codeString, {
  timeout: 5000,  // ms, optional
});

// Implementation — lazy-created hidden iframe, reused across calls:
async function evaluatePerchanceTextInSandbox(text, opts = {}) {
  let iframe = document.querySelector('#perchanceCodeEvaluationSandboxIframe');
  if(!iframe) {
    iframe = document.createElement("iframe");
    iframe.src = "https://7deabe31ae18ea5ed27c5f71b9633999.perchance.org/ai-character-chat-sandboxed-executor";
    iframe.id = "perchanceCodeEvaluationSandboxIframe";
    iframe.sandbox = "allow-scripts allow-same-origin";
    iframe.style.cssText = "position:fixed;width:1px;height:1px;opacity:0.01;top:-10px;right:-10px;pointer-events:none;border:0;";
    document.body.append(iframe);
    iframe._resolvers = {};
    let iframeLoadResolver;
    let iframeLoadPromise = new Promise(r => iframeLoadResolver = r);
    window.addEventListener('message', (event) => {
      if(event.origin !== 'https://7deabe31ae18ea5ed27c5f71b9633999.perchance.org') return;
      if(event.data.finishedLoading) { iframeLoadResolver(); return; }
      const { requestId, text } = event.data;
      if(iframe._resolvers[requestId]) {
        iframe._resolvers[requestId](text);
        delete iframe._resolvers[requestId];
      }
    });
    await iframeLoadPromise;
  }
  const requestId = Math.random().toString();
  return new Promise((resolve, reject) => {
    iframe._resolvers[requestId] = resolve;
    if(opts.timeout) {
      setTimeout(() => {
        if(iframe._resolvers[requestId]) reject("Sandbox did not respond in time.");
      }, opts.timeout);
    }
    iframe.contentWindow.postMessage({ text, requestId },
      'https://7deabe31ae18ea5ed27c5f71b9633999.perchance.org');
  });
}
```

Properties visible to custom code: `oc.character`, `oc.thread`, `oc.userCharacter`,
`oc.messages`, `oc.customData`, etc.

---

## 10 · Token Budget Management

```js
// Always check before building prompts:
const { countTokens, idealMaxContextTokens } = root.aiTextPlugin({ getMetaObject: true });
// Leave 800-token buffer — reduces prefix-cache invalidation frequency:
let budget = idealMaxContextTokens - 800;

// Character descriptions: cap at 30% of budget
if(countTokens(roleInstructionText) > budget * 0.3) {
  roleInstructionText = truncateRoleInstruction(roleInstructionText, 3000);
}

// Messages: drop oldest first until fits in remaining budget
// Keep at least 3–5 most recent exchanges
```

---

## 11 · UI Utilities

### `confirmAsync(message, opts)` — Accessible Promise-based Confirm Dialog
```js
async function confirmAsync(message, opts = {}) {
  if(!message) message = "Are you sure?";
  return new Promise(resolve => {
    const overlay = Object.assign(document.createElement("div"), { tabIndex: 0 });
    overlay.style.cssText = `position:fixed;inset:0;z-index:99999999;display:grid;place-items:center;background:rgba(0,0,0,.65);font:16px/1.4 system-ui`;
    overlay.innerHTML = `<div style="max-width:min(97vw,450px);padding:15px;border-radius:8px;background:light-dark(#fff,#222);color:light-dark(#000,#fff);">
      <p style="margin:0 0 20px;white-space:pre-wrap;">${message.replace(/[<>&]/g, m => ({"<":"&lt;","&":"&amp;",">":"&gt;"}[m]))}</p>
      <div style="display:flex;justify-content:flex-end;gap:8px;">
        <button ${opts.hideCancel ? "hidden" : ""} style="padding:6px 16px;border:1px solid light-dark(#ccc,#555);border-radius:6px;background:light-dark(#f6f6f6,#333);color:inherit;cursor:pointer;">Cancel</button>
        <button autofocus style="padding:6px 16px;border:none;border-radius:6px;background:light-dark(#1677ff,#2b87ff);color:#fff;cursor:pointer;">Okay</button>
      </div>
    </div>`;
    const [cancelBtn, okBtn] = overlay.querySelectorAll("button");
    const finish = val => { overlay.remove(); resolve(val); };
    cancelBtn.onclick = () => finish(false);
    okBtn.onclick = () => finish(true);
    overlay.onkeydown = e => {
      if(e.key === "Escape") finish(false);
      else if(e.key === "Enter") finish(true);
    };
    document.body.append(overlay);
    overlay.focus({ preventScroll: true });
  });
}
```

### `prompt2(specs, opts)` — Rich Form Modal
```js
// Standard field-based form:
let result = await window.prompt2({
  fieldName: { type: "textLine", label: "Name", placeholder: "...", defaultValue: "" },
  bio:       { type: "text",     label: "Bio",  placeholder: "..." },
  model:     { type: "select",   label: "Model", options: ["good", "great"] },
  // show(values) → bool: conditionally hide a field based on other field values
  extra:     { type: "textLine", label: "Extra", show: (values) => values.model === "great" },
}, {
  submitButtonText: "Save",
  cancelButtonText: "Cancel",  // null to hide cancel button
});
if(result) console.log(result.fieldName, result.bio);

// Raw HTML content (no form fields) — e.g. for showing a share URL:
let result = await window.prompt2({
  content: {
    type: "none",
    html: `<div style="display:flex; gap:0.5rem;">
      <input value="${shareUrl}" style="flex-grow:1;">
      <button onclick="navigator.clipboard.writeText(this.parentElement.querySelector('input').value);
        this.textContent='copied ✅'; setTimeout(()=>{ this.textContent='copy url'; }, 2000);">
        copy url
      </button>
    </div>`,
  },
}, { cancelButtonText: null, submitButtonText: "finished" });

// Available spec types: "textLine", "text", "select", "buttons", "none" (+ html)
```

### Other Modal Utilities
```js
let modal = createLoadingModal("⏳ Processing...");
modal.delete(); // remove when done

let win = createFloatingWindow({
  header: "Window Title",
  body: someHtmlElement,
  initialWidth: 400,
  initialHeight: 300,
});
```

---

## 12 · Page Initialization Pattern

```js
// Protect against load failures — show emergency export after 10s of no progress:
window.lastKnownActivelyLoadingTime = Date.now();
window.emergencyExportButtonDisplayTimeout = setInterval(() => {
  if(Date.now() - window.lastKnownActivelyLoadingTime > 10000) {
    emergencyExportCtn.hidden = false;
    initialPageLoadingModal.hidden = true;
    clearInterval(window.emergencyExportButtonDisplayTimeout);
  }
}, 5000);
// Update window.lastKnownActivelyLoadingTime during slow legit operations (e.g. lore loading)

// Standard load sequence:
// 1. Open DB
// 2. Parse URL params / hash for share link commands → checkForHashCommand()
// 3. renderThreadList()
// 4. Auto-click most recently viewed thread (or add starter character if no threads yet)
// 5. Show UI, hide loading modal
// 6. tryPersistBrowserStorageData()  ← call once, prevents browser from clearing IndexedDB
// 7. AI preload: root.aiTextPlugin({ preload: true })
// 8. Clear emergency timer: clearInterval(window.emergencyExportButtonDisplayTimeout)

// Share link / hash command routing:
let ignoreHashChange = false;
async function checkForHashCommand() {
  let urlHashJson = null;
  try { urlHashJson = JSON.parse(decodeURIComponent(window.location.hash.slice(1))); } catch(e) {}

  if(urlHashJson?.addCharacter || new URL(window.location.href).searchParams.get("data")) {
    let data = await loadDataFromShareUrl();
    let character = data?.addCharacter;
    if(character) {
      // Age-gate — warn new users about potentially sensitive content:
      let confirmed = await root.confirmAsync(
        "You've visited a character sharing link... This character may discuss sensitive themes" +
        " — please click cancel if you are under 18."
      );
      if(confirmed) {
        let result = await characterDetailsPrompt(character, {
          // quickAdd=true in URL skips the character details form:
          autoSubmit: urlHashJson.quickAdd && !editingExistingCharacter && !sameNameExists,
        });
        if(result) {
          const newCharacter = await addCharacter(result);
          await createNewThreadWithCharacterId(newCharacter.id);
        }
      }
    }
    // Clean up URL hash after processing:
    if(window.location.hash) {
      ignoreHashChange = true;
      window.location.hash = "";
      await new Promise(r => setTimeout(r, 20));
      ignoreHashChange = false;
    }
  }
}
window.addEventListener('hashchange', () => { if(!ignoreHashChange) checkForHashCommand(); });
```

### Emergency Data Export
```js
async function exportRawDb(dbName, opts = {}) {
  // Opens DB → reads all object stores → serialises to CBOR+gzip → triggers download
  // If getAll() times out for a store, falls back to one-by-one individual gets (slower but safer)
  // opts.corruptItemReplacer({storeName, id, dbData}) → return a placeholder object
  //   so corrupt records don't abort the whole export
  // opts.onProgress({message, type}) → progress callback
  // opts.skipStores → array of store names to skip
}
// Export file format: <dbName>.<timestamp>.cbor.gz
// Usage:
await exportRawDb(window.dbName, {
  onProgress: (e) => console[e.type]("exportRawDb:", e.message),
  corruptItemReplacer: ({ storeName, id, dbData }) => {
    if(storeName === "characters") return { id, name: "CORRUPT" };
    if(storeName === "threads") return { id, characterId: dbData.characters[0].id, name: "CORRUPT" };
    // returning undefined effectively deletes the corrupt item (safe for messages/lore/etc.)
  },
});
```

---

## 13 · Common Patterns

### Conditional image generation (chat messages)
```js
const imageKeywords = /\b(images?|pics?|photos?|selfie|draw|paint|generate)\b/i;
if(imageKeywords.test(fullContext)) {
  // include image syntax note in instruction
}
// AI writes: <image>detailed scene description here</image>
// Plugin renders this automatically
```

### CORS bypass (for fetching external URLs in custom code)
```js
let response = await root.superFetch("https://external.api.com/data");
let data = await response.json();
```

### Named URL characters
```js
// URL: /ai-character-chat?char=my-char-name → loads gz file from user.uploads.dev
urlNamedCharacters
  my-char-name = abc123.gz
  // NOTE: must add named chars to $meta.dynamic too (duplicate — no root.* access there)
```

### iOS Safari viewport fix (prevent auto-zoom on input focus)
```js
try {
  let isTouchScreen = window.matchMedia("(pointer: coarse)").matches;
  let isSafari = navigator.vendor?.indexOf('Apple') > -1
    && !navigator.userAgent.includes('CriOS')
    && !navigator.userAgent.includes('FxiOS');
  if(isSafari && window.innerWidth < 800 && isTouchScreen) {
    let viewportMeta = document.querySelector("[name=viewport]");
    if(!viewportMeta.getAttribute("content").includes("maximum-scale")) {
      viewportMeta.setAttribute("content",
        viewportMeta.getAttribute("content") + ", maximum-scale=1");
    }
  }
} catch(e) { console.error(e); }
```

### Persist browser storage (prevent browser clearing IndexedDB)
```js
async function tryPersistBrowserStorageData() {
  if(navigator.storage?.persist) await navigator.storage.persist();
}
// Call once after page load completes
```

### Custom modals / prompts summary
```js
// confirmAsync(message, {hideCancel}) → Promise<boolean>
// prompt2(specs, opts) → Promise<formData | null>
// createLoadingModal(text) → { delete() }
// createFloatingWindow({ header, body, initialWidth, initialHeight }) → element
```

---

## 14 · Code Review Checklist (Perchance-specific)

- [ ] All list functions use `async functionName() =>` syntax (not `function`)
- [ ] `$meta.dynamic` is fully self-contained (no `root.*` refs)
- [ ] `urlNamedCharacters` map **duplicated** inside `$meta.dynamic` — cannot use `root.*` there
- [ ] `stopSequences` always includes `"\n\n[["` for chat completions
- [ ] `hideStartWith: true` set when using `startWith` to avoid double-printing
- [ ] CORS-sensitive fetches in custom code use `superFetch`
- [ ] Token budget: `idealMaxContextTokens - 800` buffer to protect prefix cache
- [ ] `countTokens(text) > idealMaxContextTokens * 0.3` guard on role instructions
- [ ] `window.textEmbedderFunction` checked before calling `embedTexts()`
- [ ] `embedTexts()` called with `{ textArr, modelName }` object signature (not bare array)
- [ ] Share link JSON strips private user data (`id`, `lastMessageTime`, etc.)
- [ ] Sandbox origin check: `event.origin === 'https://7deabe31ae18ea5ed27c5f71b9633999.perchance.org'`
- [ ] `data.stopReason === "error"` handled before using `data.generatedText`
- [ ] Summary injection batched (`summariesReadyToInject.length >= 3` before DB write)
- [ ] `window.__aiHierarchicalSummaryStuff[threadId].alreadyDoingSummary` mutex checked
- [ ] `CompressionStream` / `DecompressionStream` availability checked before use
- [ ] `uploadPlugin` `"disallowed_content"` error handled with under-18 clarification message
- [ ] iOS Safari `maximum-scale=1` viewport patch applied for touch devices
- [ ] `tryPersistBrowserStorageData()` called once at end of page load
- [ ] Mobile preload delayed: `if(window.innerWidth < 500) setTimeout(..., 5000)`
- [ ] `$meta.dynamic` `urlNamedCharacters` comment reminder: `// NOTE: must add named chars to $meta.dynamic too`

---

## Reference Files

- `references/data-model.md` — complete DB schema with upgrade/migration patterns
- `references/message-format.md` — full message serialization spec with all edge cases
- `references/plugin-api.md` — ai-text-plugin and text-to-image-plugin extended docs

Read a reference file when you need exhaustive detail on a specific subsystem.
