<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Painel Estratégico de CX</title>

<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

</head>

<body class="bg-gray-50">

<script>
/* =========================
   CONFIGURE AQUI
========================= */
const SUPABASE_URL = 'SUA_URL_AQUI';
const SUPABASE_ANON_KEY = 'SUA_CHAVE_AQUI';

const { createClient } = supabase;
const supabaseClient = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

let currentUser = null;
let userRole = null;
let charts = {};

/* =========================
   AUTENTICAÇÃO
========================= */

window.addEventListener('DOMContentLoaded', async () => {
  const { data: { session } } = await supabaseClient.auth.getSession();
  if (session) loadUser(session.user);
});

async function login() {
  const email = document.getElementById("email").value;
  const password = document.getElementById("password").value;

  const { data, error } = await supabaseClient.auth.signInWithPassword({
    email,
    password
  });

  if (error) return alert(error.message);
  loadUser(data.user);
}

async function loadUser(user) {
  const { data, error } = await supabaseClient
    .from("users")
    .select("*")
    .eq("id", user.id)
    .single();

  if (error) return alert("Erro ao carregar usuário.");

  currentUser = data;
  userRole = data.role;

  document.getElementById("login").classList.add("hidden");
  document.getElementById("app").classList.remove("hidden");

  document.getElementById("welcome").innerText = 
    `Olá, ${data.nome}`;

  if (userRole === "colaborador") {
    document.getElementById("registroArea").classList.remove("hidden");
    loadIndividualDashboard();
  } else {
    document.getElementById("executivoArea").classList.remove("hidden");
    loadExecutiveDashboard();
  }

  enableRealtime();
}

async function logout() {
  await supabaseClient.auth.signOut();
  location.reload();
}

/* =========================
   REALTIME
========================= */

function enableRealtime() {
  supabaseClient
    .channel("realtime-registros")
    .on(
      "postgres_changes",
      { event: "*", schema: "public", table: "registros" },
      () => {
        if (userRole === "colaborador") {
          loadIndividualDashboard();
        } else {
          loadExecutiveDashboard();
        }
      }
    )
    .subscribe();
}

/* =========================
   SALVAR REGISTRO
========================= */

async function salvarRegistro() {
  const registro = {
    user_id: currentUser.id,
    tipo: document.getElementById("tipo").value,
    data: document.getElementById("data").value,
    contatos_realizados: +document.getElementById("contatos").value,
    contatos_sucesso: +document.getElementById("sucesso").value,
    detratores: +document.getElementById("detratores").value,
    solicitacoes: +document.getElementById("solicitacoes").value,
    cancelamentos: +document.getElementById("cancelamentos").value,
    retencoes: +document.getElementById("retencoes").value,
    produto: document.getElementById("produto").value,
    nicho: document.getElementById("nicho").value,
    estado: document.getElementById("estado").value,
    mrr_risco: +document.getElementById("mrrRisco").value,
    mrr_perdido: +document.getElementById("mrrPerdido").value,
    mrr_retido: +document.getElementById("mrrRetido").value
  };

  const { error } = await supabaseClient
    .from("registros")
    .insert([registro]);

  if (error) return alert(error.message);

  alert("Registro salvo com sucesso!");
}

/* =========================
   DASHBOARD INDIVIDUAL
========================= */

async function loadIndividualDashboard() {
  const { data } = await supabaseClient
    .from("registros")
    .select("*")
    .eq("user_id", currentUser.id);

  const totalContatos = sum(data, "contatos_realizados");
  const totalSucesso = sum(data, "contatos_sucesso");
  const cancelamentos = sum(data, "cancelamentos");
  const retencoes = sum(data, "retencoes");

  document.getElementById("indContatos").innerText = totalContatos;
  document.getElementById("indTaxa").innerText =
    totalContatos ? ((totalSucesso/totalContatos)*100).toFixed(1)+"%" : "0%";

  document.getElementById("indChurn").innerText =
    totalContatos ? ((cancelamentos/totalContatos)*100).toFixed(1)+"%" : "0%";
}

/* =========================
   DASHBOARD EXECUTIVO
========================= */

async function loadExecutiveDashboard() {
  const { data } = await supabaseClient
    .from("registros")
    .select("*");

  const totalContatos = sum(data,"contatos_realizados");
  const cancelamentos = sum(data,"cancelamentos");
  const mrrPerdido = sum(data,"mrr_perdido");

  document.getElementById("execContatos").innerText = totalContatos;
  document.getElementById("execChurn").innerText =
    totalContatos ? ((cancelamentos/totalContatos)*100).toFixed(1)+"%" : "0%";
  document.getElementById("execMrrPerdido").innerText =
    "R$ "+mrrPerdido.toLocaleString("pt-BR");
}

/* =========================
   UTILITÁRIOS
========================= */

function sum(data, field) {
  return data.reduce((acc, item) => acc + (item[field] || 0), 0);
}

</script>

<!-- LOGIN -->
<div id="login" class="min-h-screen flex items-center justify-center">
  <div class="bg-white p-8 rounded-xl shadow-md w-80">
    <h2 class="text-xl font-bold mb-4">Painel Estratégico CX</h2>
    <input id="email" type="email" placeholder="Email" class="w-full border p-2 mb-2 rounded">
    <input id="password" type="password" placeholder="Senha" class="w-full border p-2 mb-4 rounded">
    <button onclick="login()" class="w-full bg-blue-600 text-white py-2 rounded">
      Entrar
    </button>
  </div>
</div>

<!-- APP -->
<div id="app" class="hidden p-6">
  <div class="flex justify-between mb-6">
    <h1 class="text-2xl font-bold">Painel Estratégico de CX</h1>
    <button onclick="logout()" class="text-red-600">Sair</button>
  </div>

  <p id="welcome" class="mb-6 text-gray-600"></p>

  <!-- REGISTRO -->
  <div id="registroArea" class="hidden bg-white p-6 rounded-xl shadow mb-8">
    <h2 class="font-bold mb-4">Lançamento Diário</h2>
    <div class="grid grid-cols-2 gap-4">
      <input id="tipo" placeholder="Tipo">
      <input id="data" type="date">
      <input id="contatos" placeholder="Contatos">
      <input id="sucesso" placeholder="Sucesso">
      <input id="detratores" placeholder="Detratores">
      <input id="solicitacoes" placeholder="Solicitações">
      <input id="cancelamentos" placeholder="Cancelamentos">
      <input id="retencoes" placeholder="Retenções">
      <input id="produto" placeholder="Produto">
      <input id="nicho" placeholder="Nicho">
      <input id="estado" placeholder="Estado">
      <input id="mrrRisco" placeholder="MRR Risco">
      <input id="mrrPerdido" placeholder="MRR Perdido">
      <input id="mrrRetido" placeholder="MRR Retido">
    </div>
    <button onclick="salvarRegistro()" class="mt-4 bg-blue-600 text-white px-6 py-2 rounded">
      Salvar
    </button>
  </div>

  <!-- DASHBOARD INDIVIDUAL -->
  <div id="individualArea">
    <div class="grid grid-cols-3 gap-4">
      <div class="bg-white p-4 rounded shadow">Contatos: <span id="indContatos">0</span></div>
      <div class="bg-white p-4 rounded shadow">Taxa Sucesso: <span id="indTaxa">0%</span></div>
      <div class="bg-white p-4 rounded shadow">Churn: <span id="indChurn">0%</span></div>
    </div>
  </div>

  <!-- DASHBOARD EXECUTIVO -->
  <div id="executivoArea" class="hidden">
    <div class="grid grid-cols-3 gap-4">
      <div class="bg-white p-4 rounded shadow">Contatos: <span id="execContatos">0</span></div>
      <div class="bg-white p-4 rounded shadow">Churn: <span id="execChurn">0%</span></div>
      <div class="bg-white p-4 rounded shadow">MRR Perdido: <span id="execMrrPerdido">R$ 0</span></div>
    </div>
  </div>

</div>

</body>
</html>
