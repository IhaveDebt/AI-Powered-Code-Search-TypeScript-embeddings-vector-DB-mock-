/**
 * AI-Powered Code Search (ai_code_search.ts)
 *
 * - Simple demo: stores text "embeddings" (simulated) and performs cosine-similarity search.
 * - Replace simulateEmbedding with real embedding API (OpenAI / local) to upgrade.
 *
 * Usage:
 *   ts-node src/ai_code_search.ts seed
 *   ts-node src/ai_code_search.ts query "find binary search implementation"
 */
import fs from 'fs/promises';
import path from 'path';

const STORE = path.join(__dirname, '../ai_index.json');

type Doc = { id: string; repo: string; file: string; text: string; emb: number[] };

function tokenize(s: string) {
  return s.toLowerCase().split(/\W+/).filter(Boolean);
}
function simulateEmbedding(text: string): number[] {
  // tiny deterministic "embedding" using token counts into 64-dim vector hash
  const tokens = tokenize(text);
  const vec = new Array(64).fill(0);
  for (const t of tokens) {
    let h = 0;
    for (let i = 0; i < t.length; i++) h = (h * 31 + t.charCodeAt(i)) % 64;
    vec[h] += 1;
  }
  // normalize l2
  const norm = Math.sqrt(vec.reduce((s, v) => s + v * v, 0)) || 1;
  return vec.map(v => v / norm);
}
function cosine(a: number[], b: number[]) {
  let s = 0;
  for (let i = 0; i < a.length; i++) s += (a[i] || 0) * (b[i] || 0);
  return s;
}

async function seed() {
  const docs: Doc[] = [
    { id: 'd1', repo: 'algos', file: 'binary_search.ts', text: 'function binarySearch(arr, target) { /* ... */ }', emb: [] },
    { id: 'd2', repo: 'utils', file: 'http_client.ts', text: 'export async function fetchJson(url) { /* ... */ }', emb: [] },
    { id: 'd3', repo: 'algos', file: 'quicksort.ts', text: 'function quicksort(a) { /* ... */ }', emb: [] },
  ];
  for (const d of docs) d.emb = simulateEmbedding(d.text);
  await fs.writeFile(STORE, JSON.stringify(docs, null, 2), 'utf-8');
  console.log('Seeded', docs.length, 'docs');
}

async function query(q: string, top = 5) {
  const raw = await fs.readFile(STORE, 'utf-8');
  const docs: Doc[] = JSON.parse(raw);
  const qemb = simulateEmbedding(q);
  const scored = docs.map(d => ({ d, score: cosine(d.emb, qemb) })).sort((a, b) => b.score - a.score);
  console.log('Query:', q);
  console.log(scored.slice(0, top).map(s => ({ id: s.d.id, repo: s.d.repo, file: s.d.file, score: s.score.toFixed(4) })));
}

async function main() {
  const cmd = process.argv[2];
  if (cmd === 'seed') await seed();
  else if (cmd === 'query') await query(process.argv.slice(3).join(' '));
  else console.log('Usage: seed | query <text>');
}
main();
