<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import { supabase } from "../lib/supabase";

type Lead = {
  id: string;
  created_at: string;
  first_name: string | null;
  last_name: string | null;
  email: string;
};

const leads = ref<Lead[]>([]);
const loading = ref(true);
const error = ref<string | null>(null);
const query = ref("");
const copiedId = ref<string | null>(null);

function normalize(s: string) {
  return s.trim().toLowerCase();
}

const filtered = computed(() => {
  const q = normalize(query.value);
  if (!q) return leads.value;

  return leads.value.filter((l) => {
    const hay = `${l.email} ${l.first_name ?? ""} ${l.last_name ?? ""}`.toLowerCase();
    return hay.includes(q);
  });
});

async function loadLeads() {
  loading.value = true;
  error.value = null;

  const { data, error: err } = await supabase
    .from("leads")
    .select("id, created_at, first_name, last_name, email")
    .order("created_at", { ascending: false });

  loading.value = false;

  if (err) {
    error.value = err.message;
    return;
  }

  leads.value = data ?? [];
}

function displayName(l: Lead) {
  const fn = (l.first_name ?? "").trim();
  const ln = (l.last_name ?? "").trim();
  const name = `${fn} ${ln}`.trim();
  return name || "(no name)";
}

async function copyEmail(id: string, email: string) {
  try {
    if (window.navigator?.clipboard?.writeText) {
      await window.navigator.clipboard.writeText(email);
    } else {
      const ta = document.createElement("textarea");
      ta.value = email;
      ta.style.position = "fixed";
      ta.style.opacity = "0";
      document.body.appendChild(ta);
      ta.select();
      document.execCommand("copy");
      document.body.removeChild(ta);
    }
    copiedId.value = id;
    window.setTimeout(() => {
      if (copiedId.value === id) copiedId.value = null;
    }, 1100);
  } catch {
    // swallow; you can optionally set an error message
  }
}

defineExpose({ reload: loadLeads });

onMounted(loadLeads);
</script>

<template>
  <div class="wrap">
    <div class="toolbar">
      <button class="btn" @click="loadLeads" :disabled="loading">Refresh</button>

      <div class="search">
        <svg width="16" height="16" viewBox="0 0 24 24" class="icon" aria-hidden="true">
          <path fill="currentColor" d="M10 18a8 8 0 1 1 5.293-14.293A8 8 0 0 1 10 18m11 3-6.1-6.1a10 10 0 1 0-1.4 1.4L19.6 22z"/>
        </svg>
        <input v-model="query" placeholder="Search name or email…" />
      </div>

      <div class="count muted">{{ filtered.length }} / {{ leads.length }}</div>
    </div>

    <p v-if="loading" class="muted">Loading…</p>
    <p v-else-if="error" class="err">{{ error }}</p>

    <div v-else class="table-wrap">
      <table v-if="filtered.length" class="table">
<thead>
  <tr>
    <th class="name-col">Name</th>
    <th class="email-col">Email</th>
  </tr>
</thead>

<tbody>
  <tr v-for="l in filtered" :key="l.id">
    <td class="name name-col" data-label="Name">
      {{ displayName(l) }}
    </td>

    <td class="email-cell email-col" data-label="Email">
      <code class="mono email-text">{{ l.email }}</code>

      <button
        class="copy-btn"
        :class="{ 'is-copied': copiedId === l.id }"
        type="button"
        @click="copyEmail(l.id, l.email)"
        aria-label="Copy email"
        title="Copy email"
      >
        <svg v-if="copiedId !== l.id" class="copy-icon" viewBox="0 0 24 24" aria-hidden="true">
          <path d="M9 9h10v12H9z" />
          <path d="M5 15H4a1 1 0 0 1-1-1V4a1 1 0 0 1 1-1h10a1 1 0 0 1 1 1v1" />
        </svg>
        <svg v-else class="copy-icon" viewBox="0 0 24 24" aria-hidden="true">
          <path d="M4.5 12.5l5 5 10-10" />
        </svg>
      </button>
    </td>

  </tr>
</tbody>
</table>

      <p v-else class="muted">No leads found.</p>
    </div>
  </div>
</template>

<style scoped>
.wrap{
  display: grid;
  gap: 12px;
  width: 100%;
  max-width: none;
  margin: 0 auto;
}

.toolbar{
  display: grid;
  grid-template-columns: auto 1fr auto;
  gap: 10px;
  align-items: center;
}

.btn{
  border: 1px solid rgba(255,255,255,0.12);
  background: rgba(255,255,255,0.06);
  color: rgba(255,255,255,0.92);
  padding: 10px 12px;
  border-radius: 12px;
  cursor: pointer;
}
.btn:hover{ background: rgba(255,255,255,0.10); border-color: rgba(255,255,255,0.18); }
.btn:disabled{ opacity: 0.6; cursor: not-allowed; }

.search{
  display: flex;
  align-items: center;
  gap: 8px;
  border: 1px solid rgba(255,255,255,0.12);
  background: rgba(0,0,0,0.25);
  border-radius: 12px;
  padding: 0 10px;
}
.search input{
  width: 100%;
  border: 0;
  outline: none;
  padding: 10px 4px;
  background: transparent;
  color: rgba(255,255,255,0.92);
}
.icon{ color: rgba(255,255,255,0.55); }

.count{ font-size: 12px; }

.table-wrap{
  border: 1px solid rgba(255,255,255,0.10);
  border-radius: 14px;
  overflow-x: auto;
  overflow-y: hidden;
}

.table{
  width: max-content;
  min-width: 100%;
  border-collapse: collapse;
  table-layout: auto;
}

th, td{
  padding: 12px 12px;
  overflow: visible;
  text-overflow: clip;
  vertical-align: middle;
}

th{
  text-align: left;
}

.name-col{
  white-space: nowrap;
  min-width: 220px;
}

.email-col{
  min-width: 460px;
}

/* Email cell: allow email text to shrink & ellipsize, keep Copy visible */
.email-cell{
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
}

.email-text{
  min-width: max-content;
  flex: 0 0 auto;
  max-width: none;
  overflow: visible;
  text-overflow: clip;
  white-space: nowrap;
}


.copy-btn {
  opacity: 0;
  transition: opacity 0.15s ease, background 0.12s ease, border-color 0.12s ease, transform 0.06s ease;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex: 0 0 auto;
  width: 28px;
  height: 28px;
  padding: 0;
  border: 1px solid rgba(255,255,255,0.14);
  border-radius: 9px;
  background: rgba(255,255,255,0.03);
  color: rgba(255,255,255,0.82);
  cursor: pointer;
}

.email-cell:hover .copy-btn {
  opacity: 1;
}

.copy-btn:hover {
  background: rgba(255,255,255,0.09);
  border-color: rgba(255,255,255,0.24);
  color: rgba(255,255,255,0.95);
}

.copy-btn:active {
  transform: translateY(1px);
}

.copy-btn:focus-visible {
  outline: none;
  border-color: rgba(255,255,255,0.35);
  box-shadow: 0 0 0 2px rgba(255,255,255,0.16);
}

.copy-btn.is-copied {
  border-color: rgba(124,255,178,0.65);
  background: rgba(124,255,178,0.15);
  color: #7CFFB2;
}

.copy-icon {
  width: 15px;
  height: 15px;
  fill: none;
  stroke: currentColor;
  stroke-width: 1.9;
  stroke-linejoin: round;
  stroke-linecap: round;
}

@media (max-width: 640px) {
  .toolbar {
    grid-template-columns: auto 1fr;
  }

  .count {
    grid-column: 1 / -1;
    justify-self: end;
  }
}
</style>
