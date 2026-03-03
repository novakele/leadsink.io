<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from "vue";
import { supabase } from "./lib/supabase";
import Login from "./components/Login.vue";
import LeadsList from "./components/LeadsList.vue";
import LeadsForm from "./components/LeadsForm.vue";

const loading = ref(true);
const signedIn = ref(false);
const userEmail = ref<string | null>(null);

const leadsListRef = ref<InstanceType<typeof LeadsList> | null>(null);

let unsubscribeAuth: (() => void) | null = null;

async function refreshSession() {
  const { data } = await supabase.auth.getSession();
  signedIn.value = !!data.session;
  userEmail.value = data.session?.user?.email ?? null;
}

async function logout() {
  await supabase.auth.signOut();
  await refreshSession();
}

function handleLeadAdded() {
  leadsListRef.value?.reload();
}

function onVisibilityChange() {
  if (document.visibilityState === "visible") refreshSession();
}

onMounted(async () => {
  await refreshSession();
  loading.value = false;

  document.addEventListener("visibilitychange", onVisibilityChange);

  const { data: sub } = supabase.auth.onAuthStateChange((_event, session) => {
    signedIn.value = !!session;
    userEmail.value = session?.user?.email ?? null;
  });

  unsubscribeAuth = () => sub.subscription.unsubscribe();
});

onBeforeUnmount(() => {
  document.removeEventListener("visibilitychange", onVisibilityChange);
  unsubscribeAuth?.();
});
</script>

<template>
  <div class="page">
    <div class="container">
      <div v-if="loading" class="muted">Loading…</div>

      <Login v-else-if="!signedIn" />

      <div v-else>
        <header class="header">
          <div>
            <h1 class="title">LeadSink</h1>
            <p v-if="userEmail" class="subtitle">Signed in as <span class="mono">{{ userEmail }}</span></p>
          </div>

          <button class="btn btn-ghost" @click="logout">Log out</button>
        </header>

        <div class="grid">
          <section class="card">
            <div class="card-head">
              <h2 class="card-title">Add lead</h2>
              <p class="card-help">Quick capture for later reuse.</p>
            </div>
            <LeadsForm @lead-added="handleLeadAdded" />
          </section>

          <section class="card">
            <div class="card-head">
              <h2 class="card-title">Leads</h2>
              <p class="card-help">Search and copy emails fast.</p>
            </div>
            <LeadsList ref="leadsListRef" />
          </section>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
:root{
  --bg: #0b0d10;
  --panel: rgba(255,255,255,0.06);
  --panel2: rgba(255,255,255,0.04);
  --border: rgba(255,255,255,0.10);
  --text: rgba(255,255,255,0.92);
  --muted: rgba(255,255,255,0.65);
  --muted2: rgba(255,255,255,0.45);
  --accent: rgba(255,255,255,0.90);
  --shadow: 0 20px 60px rgba(0,0,0,0.45);
  --radius: 18px;
}

*{ box-sizing: border-box; }
html, body, #app { height: 100%; }
body{
  margin: 0;
  color: var(--text);
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji","Segoe UI Emoji";
  background:
    radial-gradient(900px 500px at 25% -20%, rgba(255,255,255,0.10), transparent 60%),
    radial-gradient(800px 400px at 90% 0%, rgba(255,255,255,0.06), transparent 55%),
    var(--bg);
}

.page{
  padding: 32px 18px 60px;
}

.container{
  max-width: 980px;
  margin: 0 auto;
}

.header{
  display:flex;
  justify-content: space-between;
  align-items:flex-start;
  gap: 16px;
  margin-bottom: 18px;
}

.title{
  margin: 0;
  font-size: 40px;
  letter-spacing: -0.03em;
  line-height: 1.05;
}

.subtitle{
  margin: 8px 0 0;
  color: var(--muted);
  font-size: 14px;
}

.mono{
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
}

.grid{
  display: grid;
  grid-template-columns: 1fr;
  gap: 14px;
}

@media (min-width: 880px){
  .grid{ grid-template-columns: 360px minmax(880px, 1fr); align-items: start; }
}

.card{
  background: linear-gradient(180deg, var(--panel), var(--panel2));
  border: 1px solid var(--border);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  padding: 16px;
  backdrop-filter: blur(10px);
}

.card-head{
  margin-bottom: 12px;
}

.card-title{
  margin: 0;
  font-size: 16px;
  letter-spacing: -0.01em;
}

.card-help{
  margin: 6px 0 0;
  color: var(--muted2);
  font-size: 13px;
}

.btn{
  border: 1px solid var(--border);
  background: rgba(255,255,255,0.06);
  color: var(--text);
  padding: 10px 12px;
  border-radius: 12px;
  cursor: pointer;
  transition: transform 0.06s ease, background 0.12s ease, border-color 0.12s ease;
}
.btn:hover{ background: rgba(255,255,255,0.10); border-color: rgba(255,255,255,0.18); }
.btn:active{ transform: translateY(1px); }
.btn:disabled{ opacity: 0.55; cursor: not-allowed; }

.btn-ghost{
  background: transparent;
}

.muted{ color: var(--muted); }
</style>