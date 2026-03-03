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

defineExpose({
  reload: loadLeads
});

function copyEmail(email: string) {
  // Clipboard API only works on HTTPS (or localhost)
  if (window.navigator?.clipboard?.writeText) {
    window.navigator.clipboard.writeText(email);
  } else {
    // Fallback for older browsers / blocked clipboard
    const ta = document.createElement("textarea");
    ta.value = email;
    ta.style.position = "fixed";
    ta.style.opacity = "0";
    document.body.appendChild(ta);
    ta.select();
    document.execCommand("copy");
    document.body.removeChild(ta);
  }
}

function normalize(s: string) {
  return s.trim().toLowerCase();
}

const filtered = computed(() => {
  const q = normalize(query.value);
  if (!q) return leads.value;

  return leads.value.filter((l) => {
    const hay = [
      l.email,
      l.first_name ?? "",
      l.last_name ?? "",
    ]
      .join(" ")
      .toLowerCase();

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

onMounted(loadLeads);
</script>

<template>
  <div style="margin-top: 24px; max-width: 900px;">
    <div style="display:flex; align-items:center; gap:12px; margin-bottom: 12px;">
      <h2 style="margin:0;">Leads</h2>
      <button @click="loadLeads" :disabled="loading">Refresh</button>

      <div style="margin-left:auto;">
        <input
          v-model="query"
          placeholder="Search name or email…"
          style="padding:10px; width: 320px;"
        />
      </div>
    </div>

    <p v-if="loading">Loading…</p>
    <p v-else-if="error" style="color:crimson;">{{ error }}</p>

    <div v-else>
      <p v-if="filtered.length === 0" style="opacity:0.8;">
        No leads found.
      </p>

      <table
        v-else
        style="width:100%; border-collapse: collapse;"
      >
        <thead>
          <tr style="text-align:left; border-bottom:1px solid #ddd;">
            <th style="padding:10px;">Name</th>
            <th style="padding:10px;">Email</th>
            <th style="padding:10px;">Created</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="l in filtered"
            :key="l.id"
            style="border-bottom:1px solid #eee;"
          >
            <td style="padding:10px;">
              <span v-if="l.first_name || l.last_name">
                {{ (l.first_name ?? "") + " " + (l.last_name ?? "") }}
              </span>
              <span v-else style="opacity:0.7;">(no name)</span>
            </td>

<td
  style="padding:10px; position:relative;"
  class="email-cell"
>
  <code>{{ l.email }}</code>

  <button
    class="copy-btn"
    @click="copyEmail(l.email)"
  >
    Copy
  </button>
</td>
            <td style="padding:10px;">
              {{ new Date(l.created_at).toLocaleString() }}
            </td>
          </tr>
        </tbody>
      </table>

      <p style="margin-top: 10px; opacity: 0.75;">
        Showing {{ filtered.length }} of {{ leads.length }}.
      </p>
    </div>
  </div>
</template>


<style scoped>
.email-cell {
  display: flex;
  align-items: center;
  gap: 8px;
}

.copy-btn {
  opacity: 0;
  transition: opacity 0.15s ease;
  font-size: 12px;
  padding: 4px 8px;
  cursor: pointer;
}

.email-cell:hover .copy-btn {
  opacity: 1;
}
</style>
