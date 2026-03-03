<script setup lang="ts">
import { ref } from "vue";
import { supabase } from "../lib/supabase";

const firstName = ref("");
const lastName = ref("");
const email = ref("");

const saving = ref(false);
const error = ref<string | null>(null);
const success = ref<string | null>(null);

async function submit() {
  error.value = null;
  success.value = null;

  const e = email.value.trim().toLowerCase();
  if (e.length >= 255) {
    error.value = "Email must be under 255 characters.";
    return;
  }

  saving.value = true;

  const { error: err } = await supabase.from("leads").insert({
    first_name: firstName.value.trim() || null,
    last_name: lastName.value.trim() || null,
    email: e,
  });

  saving.value = false;

  if (err) {
    // Unique violation will come back as an error from Postgres
    // (message varies), so we handle it generically:
    if (err.message.toLowerCase().includes("duplicate")) {
      error.value = "That email already exists.";
    } else {
      error.value = err.message;
    }
    return;
  }

  success.value = "Saved.";
  firstName.value = "";
  lastName.value = "";
  email.value = "";

  emit("lead-added");
}
</script>

<template>
  <form @submit.prevent="submit" style="display:grid; gap:10px; max-width:420px;">
    <input v-model="firstName" placeholder="First name (optional)" />
    <input v-model="lastName" placeholder="Last name (optional)" />
    <input v-model="email" placeholder="Email" type="email" required />
    <button type="submit" :disabled="saving">
      {{ saving ? "Saving..." : "Save" }}
    </button>

    <p v-if="error" style="color:crimson;">{{ error }}</p>
    <p v-if="success" style="color:green;">{{ success }}</p>
  </form>
</template>
