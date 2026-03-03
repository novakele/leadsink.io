<script setup lang="ts">
import { onMounted, ref } from "vue";
import { supabase } from "../lib/supabase";

const email = ref("");
const sent = ref(false);
const loading = ref(false);
const error = ref<string | null>(null);

// Outbound Overachiever (from RPC)
const ooLoading = ref(false);
const ooError = ref<string | null>(null);
const ooName = ref<string>("—");
const ooEmail = ref<string>("No entries yet");
const ooCopied = ref(false);

type OutboundOverachieverRow = {
  display_name: string | null;
  email: string | null;
};

function formatRecipient(name: string, email: string) {
  const cleanName = (name ?? "").trim();

  // If no real name, just return the email
  if (!cleanName || cleanName === "—" || cleanName === "(no name)") {
    return email;
  }

  // Quote names that contain special characters
  const needsQuotes = /[",]/.test(cleanName);
  const quoted = needsQuotes ? `"${cleanName.replaceAll('"', '\\"')}"` : cleanName;

  return `${quoted} <${email}>`;
}

async function copyOutboundOverachiever() {
  const emailVal = (ooEmail.value ?? "").trim();
  if (!emailVal || emailVal === "No entries yet") return;

  const textToCopy = formatRecipient(ooName.value ?? "", emailVal);

  try {
    if (window.navigator?.clipboard?.writeText) {
      await window.navigator.clipboard.writeText(textToCopy);
    } else {
      const ta = document.createElement("textarea");
      ta.value = textToCopy;
      ta.style.position = "fixed";
      ta.style.opacity = "0";
      document.body.appendChild(ta);
      ta.select();
      document.execCommand("copy");
      document.body.removeChild(ta);
    }

    ooCopied.value = true;
    window.setTimeout(() => (ooCopied.value = false), 900);
  } catch {
    // silent
  }
}

async function loadOutboundOverachiever() {
  ooLoading.value = true;
  ooError.value = null;

  // safe defaults
  ooName.value = "";
  ooEmail.value = "No overachiever yet";

  const { data, error: err } = await supabase.rpc("get_outbound_overachiever");

  ooLoading.value = false;

  if (err) {
    // Don’t scare the user on login; keep it subtle.
    ooError.value = err.message;
    return;
  }

  const rows = (data as OutboundOverachieverRow[]) ?? [];
  if (!rows.length) return;

  ooName.value = rows[0].display_name ?? "(no name)";
  ooEmail.value = rows[0].email ?? "";
}

async function sendMagicLink() {
  error.value = null;
  sent.value = false;

  const e = email.value.trim().toLowerCase();
  if (!e) {
    error.value = "Email is required.";
    return;
  }

  loading.value = true;

  const { error: err } = await supabase.auth.signInWithOtp({
    email: e,
    options: {
      // Must be allowed in Supabase Auth URL configuration
      emailRedirectTo: window.location.origin,
    },
  });

  loading.value = false;

  if (err) {
    error.value = err.message;
    return;
  }

  sent.value = true;
}

onMounted(() => {
  loadOutboundOverachiever();
});
</script>

<template>
  <div class="login">
    <div class="container">
      <div class="header">
        <div class="brand">

          <div>
            <h1 class="title">LeadSink</h1>
            <p class="subtitle">Archive unsolicited outreach. Reuse when needed.</p>
          </div>
        </div>
      </div>

      <div class="grid">
        <!-- Sign-in card -->
        <section class="card">
          <div class="card-head">
            <h2 class="card-title">Sign in</h2>
            <!-- <p class="card-help">We’ll email you a magic link.</p> -->
          </div>

          <form class="form" @submit.prevent="sendMagicLink">
            <div class="field">
              <!-- <label>Email</label> -->
              <input
                v-model="email"
                type="email"
                autocomplete="email"
                placeholder="you@company.com"
                required
              />
            </div>

            <button class="btn primary" type="submit" :disabled="loading">
              {{ loading ? "Sending…" : "Send magic link" }}
            </button>

            <p v-if="sent" class="msg ok">Check your inbox for a login link.</p>
            <p v-if="error" class="msg err">{{ error }}</p>
          </form>
        </section>

        <!-- Outbound Overachiever card -->
        <section class="card">
          <div class="card-head">
            <h2 class="card-title">Outbound Overachiever</h2>
            <!-- <button class="btn btn-ghost" type="button" @click="loadOutboundOverachiever" :disabled="ooLoading">
              {{ ooLoading ? "Loading…" : "Refresh" }}
            </button> -->
          </div>
          <p class="card-help">Excellence in unsolicited outreach.</p>

<div class="spotlight-line-single">

  <span class="spotlight-text">
    <span class="mono">{{ ooName }}</span>
    <span class="separator"> — </span>
    <span class="mono email">{{ ooEmail }}</span>
  </span>

  <button
    class="copy-btn spotlight-copy"
    @click="copyOutboundOverachiever"
  >
    <span v-if="ooCopied">Copied</span>
    <span v-else>Copy</span>
  </button>
</div>

          <p v-if="ooError" class="foot muted">
            (Couldn’t load spotlight.)
          </p>
        </section>
      </div>
    </div>
  </div>
</template>

<style scoped>
.spotlight-line-single{
  position: relative;
}


/* Keep copy button hidden by default */
.spotlight-copy{
  opacity: 0;
  transition: opacity 0.15s ease;
  margin-left: auto;
}

/* Show on hover */
.spotlight-line-single:hover .spotlight-copy{
  opacity: 1;
}

.login{
  min-height: 100vh;
  padding: 32px 18px 60px;
  display: grid;
  place-items: center;
}

.container{
  width: 100%;
  max-width: 980px;
}

.header{
  margin-bottom: 18px;
}

.brand{
  display: flex;
  align-items: center;
  gap: 14px;
}

.logo{
  width: 46px;
  height: 46px;
  border-radius: 16px;
  border: 1px solid rgba(255,255,255,0.12);
  background: rgba(255,255,255,0.06);
  display: grid;
  place-items: center;
  color: rgba(255,255,255,0.9);
  font-weight: 700;
}

.title{
  margin: 0;
  font-size: 40px;
  letter-spacing: -0.03em;
  line-height: 1.05;
}

.subtitle{
  margin: 8px 0 0;
  color: rgba(255,255,255,0.65);
  font-size: 14px;
}

.grid{
  display: grid;
  grid-template-columns: 1fr;
  gap: 14px;
}

@media (min-width: 880px){
  .grid{
    grid-template-columns: 420px 1fr;
    align-items: start;
  }
}

.card{
  background: linear-gradient(180deg, rgba(255,255,255,0.06), rgba(255,255,255,0.04));
  border: 1px solid rgba(255,255,255,0.10);
  border-radius: 18px;
  box-shadow: 0 20px 60px rgba(0,0,0,0.45);
  padding: 16px;
  backdrop-filter: blur(10px);
}

.card-head{
  margin-bottom: 12px;
}

.card-head.row{
  display: flex;
  justify-content: space-between;
  gap: 10px;
}

.card-title{
  margin: 0;
  font-size: 16px;
  letter-spacing: -0.01em;
  align-items: center;
}

.card-help{
  margin: 6px 0 0;
  color: rgba(255,255,255,0.45);
  font-size: 13px;
}

.form{
  display: grid;
  gap: 10px;
  margin-top: 10px;
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
  transition: transform 0.06s ease, background 0.12s ease, border-color 0.12s ease;
}
.btn:hover{ background: rgba(255,255,255,0.10); border-color: rgba(255,255,255,0.18); }
.btn:active{ transform: translateY(1px); }
.btn:disabled{ opacity: 0.6; cursor: not-allowed; }

.primary{ background: rgba(255,255,255,0.12); }
.btn-ghost{ background: transparent; }

.msg{ margin: 2px 0 0; font-size: 13px; }
.err{ color: #ff6b6b; }
.ok{ color: #7CFFB2; }

.spotlight{
  margin-top: 12px;
  border: 1px solid rgba(255,255,255,0.10);
  background: rgba(0,0,0,0.22);
  border-radius: 14px;
  padding: 12px;
}

.spotlight-line{
  display: grid;
  grid-template-columns: 80px 1fr;
  gap: 10px;
  padding: 8px 0;
}
.spotlight-line + .spotlight-line{
  border-top: 1px solid rgba(255,255,255,0.08);
}

.label{
  font-size: 12px;
  color: rgba(255,255,255,0.55);
  text-transform: uppercase;
  letter-spacing: 0.02em;
}

.value{
  color: rgba(255,255,255,0.92);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.mono{
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono","Courier New", monospace;
}

.foot{
  margin: 10px 0 0;
  font-size: 12px;
}
.muted{ color: rgba(255,255,255,0.55); }

.copy-btn {
  opacity: 0;
  transition: opacity 0.15s ease;
  font-size: 12px;
  padding: 4px 8px;
  cursor: pointer;
}

</style>