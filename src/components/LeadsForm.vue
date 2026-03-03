<script setup lang="ts">
import { onMounted, ref } from "vue";
import { supabase } from "../lib/supabase";

const emit = defineEmits<{ (e: "lead-added"): void }>();

const fullName = ref("");
const email = ref("");

const saving = ref(false);
const error = ref<string | null>(null);
const ok = ref<string | null>(null);

// Placeholder values (dynamic)
const namePlaceholder = ref("Dorothy Milazzo");
const emailPlaceholder = ref("name@company.com");

function splitName(name: string) {
  const parts = name.trim().split(/\s+/).filter(Boolean);

  if (parts.length === 0) return { first: null, last: null };
  if (parts.length === 1) return { first: parts[0], last: null };

  return { first: parts[0], last: parts.slice(1).join(" ") };
}

function buildFullName(first: string | null, last: string | null) {
  const fn = (first ?? "").trim();
  const ln = (last ?? "").trim();
  const combined = `${fn} ${ln}`.trim();
  return combined || null;
}

async function loadRandomPlaceholders() {
  // Defaults if no data
  namePlaceholder.value = "John Doe";
  emailPlaceholder.value = "john.doe@company.com";

  const { data, error: err } = await supabase
    .from("leads")
    .select("first_name,last_name,email")
    .limit(25);

  if (err || !data || data.length === 0) return;

  const pick = data[Math.floor(Math.random() * data.length)];
  if (!pick) return; // ✅ satisfies TS

  const full = buildFullName(pick.first_name ?? null, pick.last_name ?? null);
  if (full) namePlaceholder.value = full;

  if (pick.email) emailPlaceholder.value = pick.email;
}

async function submit() {
  error.value = null;
  ok.value = null;

  const e = email.value.trim().toLowerCase();
  if (!e) {
    error.value = "Email is required.";
    return;
  }
  if (e.length >= 255) {
    error.value = "Email must be under 255 characters.";
    return;
  }

  const { first, last } = splitName(fullName.value);

  saving.value = true;

  const { error: err } = await supabase.from("leads").insert({
    first_name: first,
    last_name: last,
    email: e,
  });

  saving.value = false;

  if (err) {
    const msg = err.message.toLowerCase();
    error.value = msg.includes("duplicate") ? "That email already exists." : err.message;
    return;
  }

  fullName.value = "";
  email.value = "";
  ok.value = "Saved.";
  emit("lead-added");

  // Optional: refresh placeholders after adding
  loadRandomPlaceholders();
}

onMounted(() => {
  loadRandomPlaceholders();
});
</script>

<template>
  <form class="form" @submit.prevent="submit">
    <div class="field">
      <label>Full name</label>
      <input
        v-model="fullName"
        :placeholder="namePlaceholder"
        autocomplete="name"
      />
    </div>

    <div class="field">
      <label>Email</label>
      <input
        v-model="email"
        :placeholder="emailPlaceholder"
        type="email"
        autocomplete="email"
        required
      />
    </div>

    <button class="btn primary" type="submit" :disabled="saving">
      {{ saving ? "Saving…" : "Save lead" }}
    </button>

    <p v-if="error" class="msg err">{{ error }}</p>
    <p v-if="ok" class="msg ok">{{ ok }}</p>
  </form>
</template>

<style scoped>
/* keep your existing nice form styles */
.form{ display:grid; gap:10px; }
.field{ display:grid; gap:6px; }
label{ font-size:12px; color: rgba(255,255,255,0.65); }
input{
  width:100%; padding:10px 12px; border-radius:12px;
  border:1px solid rgba(255,255,255,0.12);
  background: rgba(0,0,0,0.25); color: rgba(255,255,255,0.92);
  outline:none;
}
input:focus{ border-color: rgba(255,255,255,0.22); background: rgba(0,0,0,0.30); }
.btn{
  border:1px solid rgba(255,255,255,0.12);
  background: rgba(255,255,255,0.06);
  color: rgba(255,255,255,0.92);
  padding:10px 12px; border-radius:12px; cursor:pointer;
}
.btn:hover{ background: rgba(255,255,255,0.10); border-color: rgba(255,255,255,0.18); }
.btn:disabled{ opacity:0.6; cursor:not-allowed; }
.primary{ background: rgba(255,255,255,0.12); }
.msg{ margin:2px 0 0; font-size:13px; }
.err{ color:#ff6b6b; }
.ok{ color:#7CFFB2; }
</style>

<style scoped>
.form{
  display: grid;
  gap: 10px;
}

.field{
  display: grid;
  gap: 6px;
}

label{
  font-size: 12px;
  color: rgba(255,255,255,0.65);
}

input{
  width: 100%;
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid rgba(255,255,255,0.12);
  background: rgba(0,0,0,0.25);
  color: rgba(255,255,255,0.92);
  outline: none;
}
input:focus{
  border-color: rgba(255,255,255,0.22);
  background: rgba(0,0,0,0.30);
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

.primary{
  background: rgba(255,255,255,0.12);
}

.msg{
  margin: 2px 0 0;
  font-size: 13px;
}
.err{ color: #ff6b6b; }
.ok{ color: #7CFFB2; }
</style>