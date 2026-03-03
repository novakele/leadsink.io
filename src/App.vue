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
  if (document.visibilityState === "visible") {
    refreshSession();
  }
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
  <div style="padding: 24px;">
    <div v-if="loading">Loading…</div>

    <Login v-else-if="!signedIn" />

    <div v-else style="max-width: 1000px;">
      <div
        style="
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-bottom: 16px;
        "
      >
        <div>
          <h1 style="margin: 0;">LeadSink</h1>
          <p style="margin: 6px 0 0; opacity: 0.75;" v-if="userEmail">
            Signed in as {{ userEmail }}
          </p>
        </div>

        <button @click="logout">Log out</button>
      </div>

      <!-- Add Lead Form -->
      <LeadsForm @lead-added="handleLeadAdded" />

      <!-- Leads List -->
      <LeadsList ref="leadsListRef" />
    </div>
  </div>
</template>
