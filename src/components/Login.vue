<script setup lang="ts">
import { ref } from "vue";
import { supabase } from "../lib/supabase";

const email = ref("");
const sent = ref(false);
const error = ref<string | null>(null);

async function sendMagicLink() {
  error.value = null;

  const { error: err } = await supabase.auth.signInWithOtp({
    email: email.value,
    options: {
      emailRedirectTo: window.location.origin, // must be allowed in Supabase URL config
    },
  });

  if (err) error.value = err.message;
  else sent.value = true;
}
</script>

<template>
  <div style="padding:24px; max-width:420px;">
    <h1>LeadSink</h1>

    <p v-if="sent">Check your email for a login link.</p>

    <form v-else @submit.prevent="sendMagicLink">
      <input
        v-model="email"
        type="email"
        required
        placeholder="you@company.com"
        style="width:100%; padding:10px; margin-bottom:10px;"
      />
      <button type="submit">Send magic link</button>
      <p v-if="error" style="color:crimson;">{{ error }}</p>
    </form>
  </div>
</template>
