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

function formatCreated(ts: string) {
  const d = new Date(ts);
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,"0")}-${String(d.getDate()).padStart(2,"0")} ${String(d.getHours()).padStart(2,"0")}:${String(d.getMinutes()).padStart(2,"0")}`;
}

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
    }, 900);
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
    <th class="right created-col">Created</th>
  </tr>
</thead>

<tbody>
  <tr v-for="l in filtered" :key="l.id">
    <td class="name name-col" data-label="Name">
      {{ displayName(l) }}
    </td>

    <td class="email-cell email-col" data-label="Email">
      <code class="mono email-text">{{ l.email }}</code>

      <button class="copy-btn" @click="copyEmail(l.id, l.email)">
        <span v-if="copiedId === l.id">Copied</span>
        <span v-else>Copy</span>
      </button>
    </td>

    <td class="right muted created-col" data-label="Created">
      {{ formatCreated(l.created_at) }}
    </td>
  </tr>
</tbody>
</table>

      <p v-else class="muted">No leads found.</p>
    </div>
  </div>
</template>

<style scoped>
.wrap{ display: grid; gap: 12px; }

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
  overflow: hidden;
}

.table{
  width: 100%;
  border-collapse: collapse;
  table-layout: auto;
}

th, td{
  padding: 12px 12px;
  overflow: hidden;
  text-overflow: ellipsis;
}

.name-col{
  white-space: nowrap;
  width: 28%;
}

.created-col{
  white-space: nowrap;
  width: 28%;
}

.email-col{
  width: auto;         /* take remaining width */
}

/* Email cell: allow email text to shrink & ellipsize, keep Copy visible */
.email-cell{
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;        /* important for flex ellipsis */
}

.email-text{
  min-width: 0;
  flex: 1 1 auto;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}


.email-cell:hover .copy-btn {
  opacity: 1;
}

/* Existing copy button hover behavior stays */
.copy-btn {
  opacity: 0;
  transition: opacity 0.15s ease;
  font-size: 12px;
  padding: 4px 8px;
  cursor: pointer;
}
</style>