<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from "vue";
import { supabase } from "./lib/supabase";
import Login from "./components/Login.vue";

const loading = ref(true);
const signedIn = ref(false);

let unsubscribe: (() => void) | null = null;

onMounted(async () => {
  const { data } = await supabase.auth.getSession();
  signedIn.value = !!data.session;
  loading.value = false;

  const { data: sub } = supabase.auth.onAuthStateChange((_event, session) => {
    signedIn.value = !!session;
  });

  unsubscribe = () => sub.subscription.unsubscribe();
});

onBeforeUnmount(() => {
  unsubscribe?.();
});
</script>

<template>
  <div v-if="loading" style="padding:24px;">Loading...</div>

  <div v-else-if="signedIn" style="padding:24px;">
    Logged in (next: Leads UI)
  </div>

  <Login v-else />
</template>
