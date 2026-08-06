# Fixes: local Ollama / RAG / PDF pipeline (2026-08)

High-level issues found while debugging Wren against local Ollama (`llama3.2`) and Semantic Index / AI parse flows, with solutions and file references.

## 1. Semantic Index failed (`Cannot determine embedding dimension`)

**Issue:** Wren posted embeddings to `http://localhost:11434/embeddings` (404). Chat uses the native Ollama API root, but OpenAI-compatible embeddings live under `/v1/embeddings`. Switching the LLM provider to Ollama also left a stale OpenAI embedding model (`text-embedding-3-small`), so the known-dimension fallback returned nothing.

**Solution:**
- Build Ollama embed URLs as `{base}/v1/embeddings`
- Coerce incompatible embed models to `nomic-embed-text` for Ollama
- Set that default when switching provider in settings
- Include the probe error in the job failure message

**Files:**
- `src-tauri/src/rag/embeddings.rs`
- `src-tauri/src/jobs/executor.rs`
- `src/stores/settingsStore.ts`

**Ops note:** Pull an embedding model separately from chat models, e.g. `ollama pull nomic-embed-text`. `llama3.2` is not an embedder.

## 2. PDF OCR model download failed (ModelScope CDN 403)

**Issue:** ModelScope redirects to a CDN that returns 403 for default `reqwest` clients without browser-like headers.

**Solution:** Send `Referer: https://www.modelscope.cn/` and a browser `User-Agent` when downloading OCR ONNX models.

**Files:**
- `src-tauri/src/docparse/models.rs`

## 3. Document classification broke with llama3.2

**Issue:** Tool calls returned `confidence` as the string `"0.9"`. JSON-mode fallback pasted a JSON Schema into the prompt, so the model echoed the schema instead of an instance.

**Solution:**
- Lenient `f32` deserialization (accept number or numeric string)
- JSON-mode prompts use concrete instance examples, not schema dumps

**Files:**
- `src-tauri/src/llm/pipeline/classifier.rs`
- `src-tauri/src/llm/prompts.rs`

## 4. Noise detection / other JSON-mode LLM stages (schema echo)

**Issue:** Same schema-in-prompt failure for noise, discovery, and extraction JSON fallbacks. Some tool-style replies put `noise_regions` as a stringified JSON array inside a text “tool call” object.

**Solution:**
- Shared helper to append JSON instance examples for all JSON fallbacks
- Parse stringified / text tool-call noise payloads
- Recover schema-echo shapes when values are nested under `properties…`

**Files:**
- `src-tauri/src/llm/prompts.rs`
- `src-tauri/src/llm/pipeline/extractor.rs`

## 5. Structure discovery dropped most of the document

**Issue:** Weak local models treated bibliography lines as sections. When the first resolvable section was late (e.g. Conclusion), large earlier text (~12k chars) was discarded as “pre-section” content.

**Solution:**
- Skip bibliography-like discovery section names
- If the gap before the first section is > 500 chars, keep it as a `Preamble` section

**Files:**
- `src-tauri/src/llm/pipeline/discoverer.rs`
- `src-tauri/src/llm/pipeline/boundary_finder.rs`

## 6. AI-extracted year not visible in the UI

**Issue:** Year was written only to `entry_fields` (`date`), while list/card/info UI read `entries.date`.

**Solution:** Also `UPDATE entries SET date = …` when applying AI metadata year.

**Files:**
- `src-tauri/src/jobs/executor.rs`
