---
name: perchance-api
description: >
  Use this skill whenever working with Perchance.org — building generators, AI chat apps, plugins,
  or any JavaScript code that runs on perchance.org. Covers the Perchance DSL syntax, the ai-text-plugin
  API, text-to-image plugin, character/thread/message data models, IndexedDB (Dexie.js) patterns,
  hierarchical summarization, lorebooks, memory systems, streaming responses, sandboxed custom code,
  share-link/upload patterns including the hash-share fallback for forked deployments,
  upload-plugin domain-allowlist mitigation, the `expires`+`deletionUrl` upload options,
  AI runtime sentinels for letting the AI drive scene/runtime side-effects, and natural-language
  wizard skips. Trigger this skill for any mention of perchance, ai-text-plugin, perchance generator,
  ai-character-chat, or building a Perchance-hosted AI app. Also trigger when the user shows code
  that uses `root.aiTextPlugin`, `root.textToImagePlugin`, `root.uploadPlugin`, Perchance list syntax,
  the `{import:...}` pattern, `[BG:type]` / `[VIZ:type]` / `[PAUSE n]` markers, or hash-share URLs
  like `#cfg=` / `#persona=`. Trigger when seeing the upload error
  "Forbidden: This API Key is locked to specific domains".
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

The Perchance ecosystem offers two complementary share mechanisms. **Use both in cascade**: hash-share first (no server roundtrip, never expires, works on forks), upload-share when the payload is too large for a URL fragment, clipboard-JSON as a final fallback. The cascade matters because of a server-side restriction described below.

### 8.1 The domain-allowlist constraint (READ THIS FIRST)

The `upload-plugin` posts `window.generatorName` to `https://upload.perchance.org` as part of every upload request. The upload server's API key is **locked to specific generator slugs at the server side**. Forks and hex-named subdomains (e.g. `64116dfabfc9f6706198dbf7f23c4240.perchance.org`, the slug of an unsaved fork) are rejected with:

```
Forbidden: This API Key is locked to specific domains.
Current origin: https://<slug>.perchance.org.
Ensure your generator name is added to the allowed list.
```

**This is server-side. You cannot bypass it from the client.** Overriding `window.generatorName` would circumvent Perchance's security model — don't. The right mitigation is to design your share path so it doesn't require the upload server in the first place. That's what the hash-share pattern below does.

The audit-server-side fix (allowlisting your fork slug) requires intervention from Perchance's maintainers.

### 8.2 The three URL shapes

| Shape | Mechanism | Persistence | Works on forks? | Use when |
|---|---|---|---|---|
| `?data=<fileId>.gz` | `upload-plugin` → `user.uploads.dev` CDN | 30 days (default) – permanent | ❌ no (allowlist) | payload doesn't fit in URL fragment |
| `#cfg=<params>` (legacy) | URL fragment, key-value | permanent — URL is the data | ✅ yes | small structured config (preset references) |
| `#kind=<gzip-base64>` | URL fragment, opaque encoded blob | permanent — URL is the data | ✅ yes | any JSON ≤ ~3 KB after gzip |

The hash-share `#kind=<…>` is the most general. Use it as your default. Fall back to upload only when the payload exceeds the URL budget.

### 8.3 Hash-share helpers (recommended default)

```js
const HASH_SHARE_URL_BUDGET = 1900; // chars total — leaves headroom for clients that strip whitespace or add prefixes (Discord, Slack)

async function buildHashShareUrl(payload, kind) {
  try {
    const json = JSON.stringify(payload);
    // Gzip compress
    const stream = new Blob([json]).stream().pipeThrough(new CompressionStream('gzip'));
    const compressed = await new Response(stream).arrayBuffer();
    // URL-safe base64 (+ → -, / → _, no padding)
    const bytes = new Uint8Array(compressed);
    let bin = '';
    for (let i = 0; i < bytes.length; i++) bin += String.fromCharCode(bytes[i]);
    const b64 = btoa(bin).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
    const base = window.location.origin + window.location.pathname;
    const url = `${base}#${kind}=${b64}`;
    if (url.length > HASH_SHARE_URL_BUDGET) return null; // too large; caller falls through to upload
    return url;
  } catch { return null; }
}

async function decodeHashShareFragment(b64) {
  try {
    let std = b64.replace(/-/g, '+').replace(/_/g, '/');
    while (std.length % 4) std += '=';
    const bin = atob(std);
    const bytes = new Uint8Array(bin.length);
    for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
    const stream = new Blob([bytes]).stream().pipeThrough(new DecompressionStream('gzip'));
    const json = await new Response(stream).text();
    return JSON.parse(json);
  } catch { return null; }
}
```

**Budget math:** origin + pathname is typically 60-100 chars. Gzip compresses JSON 50-70%; base64 expands by 33%. Net: JSON payloads up to ~3 KB after gzip fit comfortably in 1900 chars.

### 8.4 Upload-plugin: `expires` for quota boost

The `upload-plugin` accepts an `expires` (epoch ms) option. Per its docs:

| `expires` | Quota boost |
|---|---|
| ≤ 24 hours | **400×** |
| 30 days | ~50-100× (interpolated) |
| 1 year | **20×** |
| omitted (permanent) | 1× (baseline) |

For temporary-style shares (the common case — a friend opens the link within hours/days), pass `expires: Date.now() + 30 * 86400 * 1000`. You get a meaningful quota boost in exchange for a 30-day expiry that almost no realistic share lifecycle hits.

```js
const THIRTY_DAYS_MS = 30 * 24 * 60 * 60 * 1000;
const result = await uploadPlugin(file, { expires: Date.now() + THIRTY_DAYS_MS });
```

### 8.5 Upload-plugin: `deletionUrl` capture

The upload-plugin response includes a `deletionUrl` that's valid for **3 days** after upload. After that, the deletion endpoint expires.

```js
const { url, size, error, deletionUrl } = await uploadPlugin(file, opts);
// Persist deletionUrl + createdAt in IDB so the user can revoke later:
await idbSet('shared_links', [...existing, {
  id: 'sh_' + Date.now().toString(36),
  url, deletionUrl,
  createdAt: Date.now(),
  expiresAt: opts.expires || null,
}]);

// To revoke later (within 3-day window):
const res = await fetch(deletionUrl).then(r => r.json());
// res.success will be true if deletion succeeded. Mark revoked locally
// regardless — server may cache, and re-trying is wasteful.
```

### 8.6 The three-tier cascade pattern

For any share-link feature, structure your code as:

```js
async function shareThing(payload, opts = {}) {
  // Tier 1 — hash-share (no server, permanent, works on forks)
  const hashUrl = await buildHashShareUrl(payload, opts.kind || 'data');
  if (hashUrl) {
    showShareModal(hashUrl);
    return;
  }
  // Tier 2 — upload-share (might 403 on forks; 30-day expiry by default)
  const uploadUrl = await createShareUpload(payload, {
    fileName: opts.kind || 'share',
    expires: Date.now() + THIRTY_DAYS_MS,
  });
  if (uploadUrl) {
    showShareModal(uploadUrl);
    return;
  }
  // Tier 3 — clipboard JSON (always works, manual paste)
  const accepted = await confirmAsync(
    'Couldn\'t create a share link. Copy the JSON to your clipboard so you can send it directly?'
  );
  if (accepted) await copyJsonToClipboard(payload);
}
```

**The cascade matters because hash-share NEVER fails on size-fit payloads.** The upload server is the unreliable component (allowlist + quotas + content moderation). Only hit it when you must.

### 8.7 Modal copy: distinguish the URL shapes

Users need to know whether a link will expire. Detect the URL shape and render an appropriate footer note:

```js
const isUploadShare = /\?data=/.test(url);
const isHashShare   = /#\w+=/.test(url);
let footerNote = '';
if (isUploadShare) {
  footerNote = 'Note: Upload-based share links expire after 30 days. ' +
    'For permanent sharing, send a hash-share link or use cloud upload.';
} else if (isHashShare) {
  footerNote = 'Permanent: The data is encoded in the link itself, ' +
    'so it never expires and works on any deployment — no upload server needed.';
}
```

### 8.8 `$meta.dynamic` link previews for hash-shares

`$meta.dynamic` runs in a sandboxed context with no network access, so it can't decompress the hash-share payload to extract a title. Use a generic preview based on the `kind` in the URL fragment:

```js
$meta.dynamic
  $output(params, hash) =>
    if (hash.indexOf("persona=") === 0) {
      return {
        title: "Shared Custom Guide (Persona)",
        description: "Someone shared a custom guide with you. " +
          "Open to review it and add it to your library.",
        image: defaultImage,
      };
    }
    if (hash.indexOf("cfg=") === 0) {
      // Legacy human-readable form — can parse fields directly:
      let qs = new URLSearchParams(hash.slice(4));
      return {
        title: "Shared session: " + (qs.get("m") || "custom"),
        description: "...",
        image: defaultImage,
      };
    }
    if (params.data) {
      // Upload-share — opaque from here too
      return {
        title: "Shared content",
        description: "...",
        image: defaultImage,
      };
    }
```

### 8.9 Legacy upload-only example (for reference)

The original ai-character-chat pattern, kept for reference. Use the cascade above instead unless you specifically need this legacy shape:

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

// Compression helpers (for upload payload):
async function compressBlobWithGzip(blob) {
  const cs = new CompressionStream("gzip");
  let out = await new Response(blob.stream().pipeThrough(cs)).blob();
  return new Blob([out], { type: "application/gzip" });
}
async function decompressBlobWithGzip(blob) {
  const ds = new DecompressionStream("gzip");
  return await new Response(blob.stream().pipeThrough(ds)).blob();
}

// Loading a `?data=…` upload-share:
async function loadDataFromShareUrl() {
  let dataParam = new URL(window.location.href).searchParams.get("data");
  if (!dataParam) return null;
  let fileName = dataParam.split("~").slice(-1)[0]; // strip cosmetic prefix
  let blob = await fetch("https://user.uploads.dev/file/" + fileName, {
    signal: AbortSignal.timeout?.(15000)
  }).then(r => r.ok ? r.blob() : null).catch(console.error);
  if (!blob) {
    // File may have been quarantined: perchance.org/quarantined-files
    return null;
  }
  let decompressed = await decompressBlobWithGzip(blob);
  return JSON.parse(await decompressed.text());
}
```

### 8.10 Upload-plugin error handling — observed taxonomy

```js
} catch (e) {
  const msg = String(e?.message || e);
  if (/locked to specific domains/i.test(msg) || /Forbidden/i.test(msg)) {
    // Fork / non-allowlisted deployment — fall through to hash-share
    // or clipboard. Do NOT retry.
  } else if (msg.includes('disallowed_content')) {
    // Content moderation — surface the under-18 clarification message.
  } else if (/over_daily_allowance|file_too_big|invalid_filetype/.test(msg)) {
    // Quota / shape errors — surface to user.
  } else {
    // Unknown — log and surface generic message.
  }
}
```

### 8.11 What window.generatorName is and isn't

`window.generatorName` is set by Perchance's generator-loading infrastructure. It identifies which generator slug is running. The upload server reads this to enforce the API-key allowlist. **It is not a security primitive that you should override.** Lying about it would (a) defeat the allowlist, (b) violate Perchance's terms, (c) likely break in production when the upload server adds origin verification.

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

### Natural-language wizard skip ("SmartAssist")

Multi-step wizards are friction. Let the user describe what they want in plain English; have the AI return structured JSON; dispatch each field as if the user had filled it in. Pattern:

```js
async function runSmartAssist(userCommand) {
  const instruction = buildSmartAssistPrompt(userCommand);
  // CRITICAL — `startWith: '{'` strips the opening brace from the response.
  // CRITICAL — include the chat-format-bleed safety net stops EVEN for direct
  //            aiTextPlugin calls (the aiGenerate wrapper auto-merges them,
  //            but direct callers must add manually).
  const data = await root.aiTextPlugin({
    instruction,
    startWith: '{',
    stopSequences: ['\n\n\n', '\n\n}\n', '```', '\n\n[[', '\n[['],
    hideStartWith: true,
  });
  if (!data || data.stopReason === 'error') throw new Error('AI returned an error.');
  let text = (data.generatedText || '').trim();
  if (!text) throw new Error('AI returned empty output.');
  // Re-attach the stripped opening brace + scan for matching close to defend
  // against trailing chatter:
  if (!text.startsWith('{')) text = '{' + text;
  const lb = text.lastIndexOf('}');
  if (lb > -1) text = text.substring(0, lb + 1);
  const cfg = JSON.parse(text);
  // Dispatch each field as if the user had picked it manually — fire `change`
  // events on the corresponding inputs/selects so the rest of the app's
  // wiring catches up.
  applySmartAssistConfig(cfg);
}
```

**Build the prompt by enumerating valid options inline** so the AI returns values that map cleanly:

```js
function buildSmartAssistPrompt(userCommand) {
  return `User wants: "${userCommand}"

Return a JSON object choosing values from these options:

PERSONAS (pick ONE id): ${PERSONAS.map(p => `"${p.id}" (${p.tagline})`).join(', ')}
METHODS (pick ONE id): ${METHODS.map(m => `"${m.id}" (${m.tagline})`).join(', ')}
INTENTIONS: ${Object.keys(INTENTION_TYPES).join(', ')}
LENGTHS: ${Object.keys(PHASE_DURATIONS).join(', ')}
SOUNDSCAPES: ${SOUNDSCAPES.join(', ')}

Return ONLY a JSON object like:
{"persona": "id", "method": "id", "intention": "type", "length": "size",
 "soundscape": "name", "intentionInput": "user-paraphrased intent"}

If a field is unclear from the request, omit it (don't guess). Keep
"intentionInput" to ≤120 characters.`;
}
```

**UI:** chat-style composer (textarea + accent-bordered send button on the right) so users recognize it as a chat input, not another form field. Plain Enter submits; Shift+Enter inserts a newline (chat convention).

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

### AI runtime sentinels — letting the AI direct the runtime

Sometimes you want the AI to drive the runtime, not just emit text. Example: in a hypnosis app, when the AI writes "let the rain wash through your shoulders", the soundscape *should* be rain at that moment. Hardcoding rain in the wizard is wrong (the user picked something else); ignoring the language is also wrong (jarring mismatch).

**Pattern:** let the AI emit bracketed directives in its output that get parsed before display, dispatched to the runtime, and stripped from text. Same shape as `[PAUSE n]` markers but for any side-effect:

```js
// Strict allowlist — the AI cannot drive the runtime into invalid states
const VALID_BG_VALUES = new Set(['rain', 'ocean', 'fire', /* … */]);
const VALID_VIZ_VALUES = new Set(['spiral', 'tunnel', 'aurora', /* … */]);

function extractSceneSentinels(text) {
  const dispatches = [];
  const stripped = text.replace(/\[(BG|VIZ):\s*([\w\-]+)\s*\]/gi, (_, kind, value) => {
    const k = kind.toUpperCase();
    const v = String(value || '').toLowerCase().trim();
    if (k === 'BG' && VALID_BG_VALUES.has(v))   dispatches.push({ kind: 'bg', value: v });
    if (k === 'VIZ' && VALID_VIZ_VALUES.has(v)) dispatches.push({ kind: 'viz', value: v });
    return ''; // ALWAYS strip from text — never leak into TTS
  });
  return { stripped, dispatches };
}

function dispatchSceneSentinels(dispatches) {
  let delay = 0;
  for (const d of dispatches) {
    setTimeout(() => switchSceneLive(d.kind, d.value), delay);
    delay += 800; // gap so audio engine doesn't churn
  }
}
```

**Five non-obvious rules:**

1. **Always strip, even when disabled.** A "disable AI scene directives" toggle should disable the *dispatch*, not the strip. Sentinels must never leak into TTS as "open bracket BG colon rain close bracket".
2. **Strict allowlist, silent drop.** AI output is non-deterministic. `[BG:waterfall]` — drop it silently. Don't throw; don't log noisily.
3. **Inter-dispatch gap.** Audio/visual engines take time to start/stop. Stacked directives like `[BG:rain][VIZ:waveform][BG:ocean]` need ~800ms between dispatches so engines don't fight each other.
4. **Fire and forget.** The dispatch is a side-effect on the runtime; don't `await` it inside the speech path. Speech should not block on engine churn.
5. **Prompt the AI sparingly.** Add the sentinel primer to your shared context as a *low-priority* block (e.g. priority 2 in your token-budget hierarchy) so it trims out under tight context. List the valid values explicitly, instruct "use sparingly, prefer at phase start, skip when current scene already fits".

**Hooking in:** put the extraction BEFORE your generic bracket-strip pass. The order matters because the generic strip would erase your sentinels if it ran first.

```js
// In speakScript or equivalent text-to-speech handler:
canonical = canonical.replace(/\[\[(.+?)\]\]/g, '\u0001$1\u0002'); // protect [[ ]] markers
const ext = extractSceneSentinels(canonical);
canonical = ext.stripped;
if (state.aiSceneDirectivesEnabled !== false) dispatchSceneSentinels(ext.dispatches);
canonical = canonical.replace(/\[[^\]]+\]/g, m =>
  /^\[PAUSE\s+\d+\]$/i.test(m) ? m : ''
); // generic strip — preserves PAUSE, erases everything else
```

This pattern generalizes beyond audio/visuals — any runtime side-effect the AI could meaningfully direct (camera angles in a 3D app, lighting in a scene generator, character moods in an RPG) fits the same shape.

---

## 14 · Code Review Checklist (Perchance-specific)

- [ ] All list functions use `async functionName() =>` syntax (not `function`)
- [ ] `$meta.dynamic` is fully self-contained (no `root.*` refs)
- [ ] `urlNamedCharacters` map **duplicated** inside `$meta.dynamic` — cannot use `root.*` there
- [ ] `stopSequences` always includes `"\n\n[["` for chat completions
- [ ] **Direct `root.aiTextPlugin` callers ALSO include `"\n[["`** — chat-format-bleed shows up with single newline too. The `aiGenerate` wrapper auto-merges both; direct callers must add manually.
- [ ] `hideStartWith: true` set when using `startWith` to avoid double-printing
- [ ] **`startWith: '{'` callers re-attach the brace + scan for matching `}` to defend against trailing chatter** — the option strips the opening brace from the response
- [ ] CORS-sensitive fetches in custom code use `superFetch`
- [ ] Token budget: `idealMaxContextTokens - 800` buffer to protect prefix cache
- [ ] `countTokens(text) > idealMaxContextTokens * 0.3` guard on role instructions
- [ ] `window.textEmbedderFunction` checked before calling `embedTexts()`
- [ ] `embedTexts()` called with `{ textArr, modelName }` object signature (not bare array)
- [ ] Share link JSON strips private user data (`id`, `lastMessageTime`, etc.)
- [ ] **Share path uses the three-tier cascade** — hash-share → upload → clipboard JSON. Hash-share is preferred (no server roundtrip, never expires, works on forks).
- [ ] **Upload calls pass `expires`** — 30 days default for ~50-100× quota boost. Permanent retention is rarely worth the quota tax.
- [ ] **`deletionUrl` from upload response is captured + persisted** — IDB ring buffer per-share with 3-day revoke window. Don't strand it in console logs.
- [ ] **Upload errors are matched specifically** — `/locked to specific domains/i` (allowlist), `disallowed_content` (moderation), `over_daily_allowance|file_too_big|invalid_filetype` (quota/shape). Each gets a different user-facing message.
- [ ] **DON'T override `window.generatorName`** to bypass the upload-plugin allowlist. Use hash-share fallback instead.
- [ ] `$meta.dynamic` covers all share URL shapes — `?data=…`, `#cfg=…`, `#kind=…` — with appropriate generic previews when the payload is opaque.
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
