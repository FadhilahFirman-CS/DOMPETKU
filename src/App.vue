<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from "vue";
import { Chart, LineController, LineElement, PointElement, LinearScale, CategoryScale, Filler, Tooltip, BarController, BarElement } from "chart.js";
import { gsap } from "gsap";
import Lenis from "lenis";
import {
  ArrowDownLeft, ArrowUpRight, BarChart3, Bell, CalendarDays, ChevronRight,
  CircleDollarSign, Download, Home, LayoutDashboard, Menu, PiggyBank,
  Plus, Settings, ShoppingBag, Sparkles, Target, TrendingDown, TrendingUp,
  Upload, Wallet, X,
} from "lucide-vue-next";

Chart.register(LineController, LineElement, PointElement, LinearScale, CategoryScale, Filler, Tooltip, BarController, BarElement);

type View = "overview" | "transactions" | "budgets" | "reports" | "settings";
type Transaction = { id: number; name: string; category: string; date: string; amount: number; type: "income" | "expense" };
type Goal = { id: number; name: string; target: number; saved: number; deadline: string };

const view = ref<View>("overview");
const isLoading = ref(true);
const loadingProgress = ref(0);
const mobileOpen = ref(false);
const showNotifications = ref(false);
const showRangeMenu = ref(false);
const selectedRange = ref(6);
const periodIndex = ref(3);
const now = ref(new Date());
const showModal = ref(false);
const showGoalModal = ref(false);
const goalDraft = ref({ name: "", target: "", saved: "", deadline: "" });
const showFundModal = ref(false);
const selectedGoal = ref<Goal | null>(null);
const fundAmount = ref("");
const chartCanvas = ref<HTMLCanvasElement | null>(null);
const reportCanvas = ref<HTMLCanvasElement | null>(null);
const form = ref({ name: "", category: "Lainnya", amount: "", type: "income" as Transaction["type"] });
const storageKey = "dompetku_transactions_v1";
const goalsKey = "dompetku_goals_v1";
const seed: Transaction[] = [
  { id: 1, name: "Gaji bulanan", category: "Gaji", date: "2024-05-28", amount: 12500000, type: "income" },
  { id: 2, name: "Belanja bulanan", category: "Belanja", date: "2024-05-26", amount: 1450000, type: "expense" },
  { id: 3, name: "Kopi & makan siang", category: "Makanan", date: "2024-05-24", amount: 85000, type: "expense" },
  { id: 4, name: "Freelance project", category: "Lainnya", date: "2024-05-21", amount: 2750000, type: "income" },
  { id: 5, name: "Tagihan internet", category: "Tagihan", date: "2024-05-18", amount: 350000, type: "expense" },
];
const transactions = ref<Transaction[]>(JSON.parse(localStorage.getItem(storageKey) || "null") || seed);
const goals = ref<Goal[]>(JSON.parse(localStorage.getItem(goalsKey) || "null") || []);

const navigation = [
  { id: "overview" as View, label: "Overview", icon: LayoutDashboard },
  { id: "transactions" as View, label: "Transaksi", icon: CircleDollarSign },
  { id: "budgets" as View, label: "Budget", icon: PiggyBank },
  { id: "reports" as View, label: "Laporan", icon: BarChart3 },
  { id: "settings" as View, label: "Pengaturan", icon: Settings },
];
const income = computed(() => transactions.value.filter((item) => item.type === "income").reduce((sum, item) => sum + item.amount, 0));
const expense = computed(() => transactions.value.filter((item) => item.type === "expense").reduce((sum, item) => sum + item.amount, 0));
const balance = computed(() => income.value - expense.value);
const savingsRate = computed(() => income.value ? Math.round((balance.value / income.value) * 100) : 0);
const categoryTotals = computed(() => {
  const totals = transactions.value.filter((item) => item.type === "expense").reduce<Record<string, number>>((result, item) => { result[item.category] = (result[item.category] || 0) + item.amount; return result; }, {});
  return Object.entries(totals).sort((a, b) => b[1] - a[1]);
});
const topCategory = computed(() => categoryTotals.value[0]?.[0] || "Belum ada");
const budgetTotal = computed(() => income.value);
const budgetUsed = computed(() => expense.value);
const budgetPercent = computed(() => budgetTotal.value ? Math.min(100, Math.round((budgetUsed.value / budgetTotal.value) * 100)) : 0);
const periodOptions = [{ label: "1 Minggu", months: 1 }, { label: "1 Bulan", months: 1 }, { label: "3 Bulan", months: 3 }, { label: "6 Bulan", months: 6 }];
const periodLabel = computed(() => periodOptions[periodIndex.value].label);
const selectPeriod = (index: number) => { periodIndex.value = index; selectedRange.value = periodOptions[index].months; const button = document.querySelector<HTMLButtonElement>(".chart-panel .select-button"); if (button?.firstChild) button.firstChild.textContent = `${periodLabel.value} `; document.querySelector(".range-menu-floating")?.remove(); };
const handlePeriodClick = (event: Event) => {
  const target = event.target as HTMLElement;
  const button = target.closest<HTMLButtonElement>(".chart-panel .select-button");
  if (!button) return;
  document.querySelector(".range-menu-floating")?.remove();
  const menu = document.createElement("div"); menu.className = "range-menu-floating";
  periodOptions.forEach((option, index) => { const item = document.createElement("button"); item.type = "button"; item.textContent = option.label; if (index === periodIndex.value) item.classList.add("selected"); item.addEventListener("click", () => selectPeriod(index)); menu.appendChild(item); });
  const rect = button.getBoundingClientRect(); menu.style.top = `${rect.bottom + 8}px`; menu.style.left = `${rect.right - 150}px`; document.body.appendChild(menu);
};
const handleBudgetClick = (event: Event) => { const target = event.target as HTMLElement; const button = target.closest(".view-section .page-heading .primary-button"); if (button && button.textContent?.includes("Tambah budget")) showModal.value = true; };
const rupiah = (value: number) => new Intl.NumberFormat("id-ID", { style: "currency", currency: "IDR", maximumFractionDigits: 0 }).format(value).replace("IDR", "Rp");
const dateLabel = (value: string) => new Intl.DateTimeFormat("id-ID", { day: "2-digit", month: "short", year: "numeric" }).format(new Date(value));

let chart: Chart | undefined;
let lenis: Lenis | undefined;
let clock: ReturnType<typeof setInterval> | undefined;
const currentDateTime = computed(() => new Intl.DateTimeFormat("id-ID", { day: "2-digit", month: "short", year: "numeric", hour: "2-digit", minute: "2-digit" }).format(now.value));
const save = () => localStorage.setItem(storageKey, JSON.stringify(transactions.value));
const saveGoals = () => localStorage.setItem(goalsKey, JSON.stringify(goals.value));
const addGoal = () => { goalDraft.value = { name: "", target: "", saved: "", deadline: "" }; showGoalModal.value = true; };
const saveGoal = () => { const target = Number(goalDraft.value.target); if (!goalDraft.value.name.trim() || !target || target < 1) return; goals.value.push({ id: Date.now(), name: goalDraft.value.name.trim(), target, saved: Math.max(0, Number(goalDraft.value.saved) || 0), deadline: goalDraft.value.deadline || "Tanpa deadline" }); saveGoals(); showGoalModal.value = false; };
const addGoalFunds = (goal: Goal) => { selectedGoal.value = goal; fundAmount.value = ""; showFundModal.value = true; };
const saveGoalFunds = () => { const amount = Number(fundAmount.value); if (!selectedGoal.value || !amount || amount < 1) return; selectedGoal.value.saved += amount; saveGoals(); showFundModal.value = false; selectedGoal.value = null; };
const goalPercent = (goal: Goal) => Math.min(100, Math.round((goal.saved / goal.target) * 100));
const exportData = () => undefined;
const selectView = (next: View) => { view.value = next; mobileOpen.value = false; window.scrollTo({ top: 0, behavior: "smooth" }); };
const addTransaction = () => {
  if (!form.value.name || !form.value.amount) return;
  transactions.value.unshift({ id: Date.now(), name: form.value.name, category: form.value.category, date: new Date().toISOString().slice(0, 10), amount: Number(form.value.amount), type: form.value.type });
  save(); form.value = { name: "", category: "Lainnya", amount: "", type: "income" }; showModal.value = false;
};
const removeTransaction = (id: number) => { transactions.value = transactions.value.filter((item) => item.id !== id); save(); };
const renderChart = () => {
  if (!chartCanvas.value || view.value !== "overview") return;
  chart?.destroy();
  const labels = ["Jul", "Agu", "Sep", "Okt", "Nov", "Des", "Jan", "Feb", "Mar", "Apr", "Mei", "Jun"].slice(-selectedRange.value);
  const incomeData = [7.4, 8.1, 9.3, 10.1, 11.2, 10.7, 8.2, 10.5, 9.6, 12.8, 11.4, income.value / 1000000].slice(-selectedRange.value);
  const expenseData = [4.8, 5.1, 5.4, 5.8, 6.3, 5.9, 5.1, 6.2, 5.8, 7.3, 6.1, expense.value / 1000000].slice(-selectedRange.value);
  chart = new Chart(chartCanvas.value, { type: "line", data: { labels, datasets: [
    { label: "Pemasukan", data: incomeData, borderColor: "#12b8b9", backgroundColor: "rgba(18,184,185,.14)", fill: true, tension: .42, pointRadius: 0, borderWidth: 3 },
    { label: "Pengeluaran", data: expenseData, borderColor: "#ff8a00", backgroundColor: "transparent", tension: .42, pointRadius: 0, borderWidth: 2 },
  ] }, options: { responsive: true, maintainAspectRatio: false, plugins: { legend: { display: false }, tooltip: { displayColors: false } }, scales: { x: { grid: { display: false }, ticks: { color: "#8f96b5" } }, y: { grid: { color: "rgba(255,255,255,.06)" }, ticks: { color: "#8f96b5", callback: (value) => `Rp${value}jt` } } } } });
};
const renderReportChart = () => {
  if (!reportCanvas.value || view.value !== "reports") return;
  chart?.destroy();
  chart = new Chart(reportCanvas.value, { type: "bar", data: { labels: ["Jan", "Feb", "Mar", "Apr", "Mei", "Jun"], datasets: [
    { label: "Pemasukan", data: [8.2, 10.5, 9.6, 12.8, 11.4, income.value / 1000000], backgroundColor: "#12b8b9", borderRadius: 7, barThickness: 18 },
    { label: "Pengeluaran", data: [5.1, 6.2, 5.8, 7.3, 6.1, expense.value / 1000000], backgroundColor: "#ff8a00", borderRadius: 7, barThickness: 18 },
  ] }, options: { responsive: true, maintainAspectRatio: false, plugins: { legend: { labels: { color: "#aab1cf", usePointStyle: true, padding: 18 } }, tooltip: { callbacks: { label: (context) => ` Rp${context.raw}jt` } } }, scales: { x: { grid: { display: false }, ticks: { color: "#8f96b5" } }, y: { grid: { color: "rgba(255,255,255,.06)" }, ticks: { color: "#8f96b5", callback: (value) => `Rp${value}jt` } } } } });
};
onMounted(async () => {
  lenis = new Lenis({ duration: .8, smoothWheel: true, syncTouch: false });
  const raf = (time: number) => { lenis?.raf(time); requestAnimationFrame(raf); }; requestAnimationFrame(raf);
  await nextTick();
  renderChart();
  document.addEventListener("click", handlePeriodClick);
  document.addEventListener("click", handleBudgetClick);
  gsap.to(loadingProgress, { value: 100, duration: 1.65, ease: "power2.inOut", onComplete: () => {
    isLoading.value = false;
    nextTick(() => {
      const puzzlePieces = ".sidebar, .topbar, .hero-copy > *, .stat-card, .dashboard-grid > *, .transactions-panel";
      gsap.from(puzzlePieces, {
        autoAlpha: 0,
        y: (index) => index % 2 ? 28 : -22,
        x: (index) => index % 3 === 0 ? -20 : index % 3 === 1 ? 16 : 0,
        rotation: (index) => index % 2 ? 1.5 : -1.5,
        scale: .94,
        duration: .7,
        stagger: { each: .1, from: "start" },
        ease: "back.out(1.35)",
        clearProps: "all",
      });
    });
  } });
  clock = setInterval(() => { now.value = new Date(); }, 1000);
});
watch([view, selectedRange], async () => { await nextTick(); if (view.value === "reports") renderReportChart(); else renderChart(); gsap.from(".view-enter > *", { y: 12, autoAlpha: 0, duration: .42, stagger: .04, ease: "power2.out", clearProps: "all" }); });
onUnmounted(() => { chart?.destroy(); lenis?.destroy(); if (clock) clearInterval(clock); document.removeEventListener("click", handlePeriodClick); document.removeEventListener("click", handleBudgetClick); });
</script>

<template>
  <Transition name="loading-fade">
    <div v-if="isLoading" class="loading-screen">
      <div class="loading-orbit orbit-one"></div><div class="loading-orbit orbit-two"></div>
      <div class="loading-glow glow-one"></div><div class="loading-glow glow-two"></div>
      <div class="loading-content">
        <div class="loading-visual"><div class="loading-ring ring-back"></div><div class="loading-ring ring-front"></div><img class="loading-center-logo" src="/Logo_DOMPETKU.png" alt="Dompetku"/><span class="pulse-dot dot-a"></span><span class="pulse-dot dot-b"></span></div>
        <p class="loading-kicker">MENYIAPKAN RUANG FINANSIALMU</p>
        <h2>Rapiin uangmu,<br/><em>bebasin pikiranmu.</em></h2>
        <div class="loading-status"><span>Menyusun dashboard</span><strong>{{ Math.round(loadingProgress) }}%</strong></div>
        <div class="loading-bar"><span :style="{ width: `${loadingProgress}%` }"></span></div>
        <div class="loading-tips"><Sparkles :size="14"/> Setiap langkah kecil membangun masa depan yang lebih tenang.</div>
      </div>
    </div>
  </Transition>
  <div v-if="!isLoading" class="app-shell">
    <aside class="sidebar" :class="{ open: mobileOpen }">
      <div class="brand"><img class="brand-logo" src="/Logo_DOMPETKU.png" alt="Dompetku" /><span class="brand-wordmark">dompetku</span></div>
      <nav class="nav-list">
        <button v-for="item in navigation" :key="item.id" class="nav-item" :class="{ active: view === item.id }" @click="selectView(item.id)"><component :is="item.icon" :size="18" /><span>{{ item.label }}</span><b v-if="item.id === 'transactions'">{{ transactions.length }}</b></button>
      </nav>
      <div class="sidebar-bottom"><button class="nav-item" @click="selectView('settings')"><Settings :size="18" /><span>Pengaturan</span></button></div>
    </aside>
    <main class="main-content">
      <header class="topbar"><button class="mobile-menu" @click="mobileOpen = !mobileOpen"><Menu v-if="!mobileOpen"/><X v-else/></button><div class="breadcrumb"><span>Workspace</span><ChevronRight :size="14"/><strong>{{ navigation.find((item) => item.id === view)?.label }}</strong></div><div class="top-actions"><div class="notification-wrap"><button class="icon-button" :class="{ active: showNotifications }" aria-label="Buka notifikasi" @click="showNotifications = !showNotifications"><Bell :size="17"/><i></i></button><Transition name="dropdown"><div v-if="showNotifications" class="notification-popover"><div class="notification-heading"><strong>Notifikasi</strong><span>1 baru</span></div><div class="notification-item"><span class="notification-badge"><TrendingUp :size="15"/></span><div><strong>Cashflow tetap positif</strong><p>Keuanganmu terlihat sehat hari ini.</p><small>Baru saja</small></div></div><button class="notification-footer" @click="showNotifications = false">Tutup</button></div></Transition></div><button class="date-selector" aria-label="Tanggal dan waktu saat ini"><CalendarDays :size="16"/> {{ currentDateTime }}</button></div></header>
      <div class="page-content view-enter" :key="view">
        <section v-if="view === 'overview'" class="overview-view">
          <div class="page-heading hero-copy"><div><span class="eyebrow">SELAMAT DATANG KEMBALI, ANDI</span><h1>Rapiin uangmu,<br/><em>bebasin pikiranmu.</em></h1><p class="muted">Pantau pemasukan, pengeluaran, dan target finansialmu dalam satu ruang yang lebih tenang.</p></div><button class="primary-button" @click="showModal = true"><Plus :size="17"/>Tambah transaksi</button></div>
          <section class="stats-grid"><article class="stat-card balance-card"><div class="stat-top"><span class="stat-label">TOTAL SALDO</span><span class="stat-icon"><Wallet :size="19"/></span></div><div class="stat-value">{{ rupiah(balance) }}</div><div class="stat-meta positive"><TrendingUp :size="16"/>0% dari bulan lalu</div></article><article class="stat-card"><div class="stat-top"><span class="stat-label">PEMASUKAN</span><span class="stat-icon mint"><ArrowDownLeft :size="19"/></span></div><div class="stat-value">{{ rupiah(income) }}</div><div class="stat-meta positive"><TrendingUp :size="16"/>12.5% dari bulan lalu</div><div class="stat-progress"><span style="width:72%"></span></div></article><article class="stat-card"><div class="stat-top"><span class="stat-label">PENGELUARAN</span><span class="stat-icon coral"><ShoppingBag :size="19"/></span></div><div class="stat-value">{{ rupiah(expense) }}</div><div class="stat-meta negative"><TrendingDown :size="16"/>5.8% dari bulan lalu</div><div class="stat-progress coral-progress"><span style="width:52%"></span></div></article></section>
          <section class="dashboard-grid"><article class="panel chart-panel"><div class="panel-heading"><div><h2>Cashflow</h2><p class="muted">Pemasukan dan pengeluaran bulan ini</p></div><button class="select-button">6 Bulan <ChevronRight :size="15"/></button></div><div class="chart-wrap"><canvas ref="chartCanvas"></canvas></div></article><article class="panel goals-panel"><div class="panel-heading"><div><h2>Target finansial</h2><p class="muted">Sedikit demi sedikit, jadi bukit.</p></div><Target :size="22"/></div><div class="goal-list"><div class="goal-item"><div class="goal-icon blue"><PlaneIcon /></div><div><div class="goal-title"><strong>Liburan ke Bali</strong><span>68%</span></div><div class="goal-track"><span style="width:68%"></span></div><small>Rp6.800.000 / Rp10.000.000</small></div></div><div class="goal-item"><div class="goal-icon purple"><Home :size="18"/></div><div><div class="goal-title"><strong>Rumah pertama</strong><span>32%</span></div><div class="goal-track purple-track"><span style="width:32%"></span></div><small>Rp32.000.000 / Rp100.000.000</small></div></div></div><button class="add-goal" @click="selectView('budgets')"><Plus :size="15"/>Tambah tujuan baru</button></article></section>
          <section class="panel transactions-panel"><div class="panel-heading"><div><h2>Riwayat transaksi</h2><p class="muted">Aktivitas finansial terbarumu</p></div><button class="outline-button" @click="selectView('transactions')">Lihat semua <ArrowUpRight :size="15"/></button></div><div class="transaction-table-wrap"><table><thead><tr><th>TRANSAKSI</th><th>KATEGORI</th><th>TANGGAL</th><th>JUMLAH</th></tr></thead><tbody><tr v-for="item in transactions.slice(0,5)" :key="item.id"><td><div class="trans-name"><span class="trans-icon" :class="item.type"><ArrowDownLeft v-if="item.type === 'income'" :size="15"/><ShoppingBag v-else :size="15"/></span>{{ item.name }}</div></td><td>{{ item.category }}</td><td>{{ dateLabel(item.date) }}</td><td :class="item.type === 'income' ? 'amount-income' : 'amount-expense'">{{ item.type === 'income' ? '+' : '-' }} {{ rupiah(item.amount) }}</td></tr></tbody></table></div></section>
        </section>
        <section v-else-if="view === 'transactions'" class="view-section"><div class="page-heading"><div><span class="eyebrow">ACTIVITY</span><h1>Semua transaksi.</h1><p class="muted">Kelola arus uangmu dengan lebih sadar.</p></div><div class="heading-actions"><button class="outline-button" @click="exportData"><Download :size="16"/>Export</button><button class="primary-button" @click="showModal = true"><Plus :size="16"/>Tambah transaksi</button></div></div><article class="panel transactions-panel"><div class="transaction-table-wrap"><table><thead><tr><th>TRANSAKSI</th><th>KATEGORI</th><th>TANGGAL</th><th>JUMLAH</th><th></th></tr></thead><tbody><tr v-for="item in transactions" :key="item.id"><td><div class="trans-name"><span class="trans-icon" :class="item.type"><ArrowDownLeft v-if="item.type === 'income'" :size="15"/><ShoppingBag v-else :size="15"/></span>{{ item.name }}</div></td><td>{{ item.category }}</td><td>{{ dateLabel(item.date) }}</td><td :class="item.type === 'income' ? 'amount-income' : 'amount-expense'">{{ item.type === 'income' ? '+' : '-' }} {{ rupiah(item.amount) }}</td><td><button class="row-menu" @click="removeTransaction(item.id)"><X :size="15"/></button></td></tr></tbody></table></div></article></section>
<section v-else-if="view === 'budgets'" class="view-section"><div class="page-heading"><div><span class="eyebrow">PLANNING</span><h1>Target finansialmu.</h1><p class="muted">Buat tujuan yang benar-benar ingin kamu capai.</p></div><button class="primary-button" @click="addGoal"><Plus :size="16"/>Tambah target</button></div><div v-if="!goals.length" class="panel empty-goals"><Target :size="30"/><h2>Belum ada target</h2><p class="muted">Mulai dari satu tujuan kecil, lalu bangun konsisten.</p><button class="primary-button" @click="addGoal"><Plus :size="16"/>Buat target pertama</button></div><div v-else class="budget-grid goal-grid"><article v-for="goal in goals" :key="goal.id" class="panel budget-card"><div class="goal-icon blue"><Target :size="19"/></div><h3>{{ goal.name }}</h3><p class="muted">Deadline {{ goal.deadline }}</p><div class="goal-track"><span :style="{ width: `${goalPercent(goal)}%` }"></span></div><strong>{{ goalPercent(goal) }}% terkumpul</strong><small>{{ rupiah(goal.saved) }} / {{ rupiah(goal.target) }}</small><button class="goal-fund-button" @click="addGoalFunds(goal)"><Plus :size="13"/>Tambah dana</button></article></div></section>
        <section v-else-if="view === 'reports'" class="view-section"><div class="page-heading"><div><span class="eyebrow">INSIGHTS</span><h1>Baca pola uangmu.</h1><p class="muted">Ringkasan nyata dari transaksi yang sudah kamu catat.</p></div></div><div class="report-summary"><article class="panel report-metric"><span class="stat-label">SALDO BERSIH</span><strong>{{ rupiah(balance) }}</strong><small :class="balance >= 0 ? 'positive' : 'negative'"><TrendingUp :size="14"/> {{ balance >= 0 ? 'Cashflow positif' : 'Perlu perhatian' }}</small></article><article class="panel report-metric mint-card"><span class="stat-label">SAVING RATE</span><strong>{{ savingsRate }}%</strong><small class="positive"><PiggyBank :size="14"/> Dari total pemasukan</small></article><article class="panel report-metric coral-card"><span class="stat-label">KATEGORI TERBESAR</span><strong>{{ topCategory }}</strong><small class="negative"><ShoppingBag :size="14"/> {{ rupiah(categoryTotals[0]?.[1] || 0) }}</small></article></div><div class="report-layout"><article class="panel report-chart-panel"><div class="panel-heading"><div><h2>Perbandingan cashflow</h2><p class="muted">Pemasukan dan pengeluaran per bulan</p></div><BarChart3 :size="22"/></div><div class="report-chart-wrap"><canvas ref="reportCanvas"></canvas></div></article><article class="panel category-panel"><div class="panel-heading"><div><h2>Pengeluaran per kategori</h2><p class="muted">Bulan berjalan</p></div><ShoppingBag :size="20"/></div><div class="category-list"><div v-for="([category, total], index) in categoryTotals" :key="category" class="category-row"><div class="category-row-top"><strong>{{ category }}</strong><span>{{ rupiah(total) }}</span></div><div class="category-track"><span :style="{ width: `${Math.max(10, (total / (expense || 1)) * 100)}%`, background: index % 2 ? '#9c7aff' : '#56dfb5' }"></span></div></div><p v-if="!categoryTotals.length" class="muted">Belum ada data pengeluaran.</p></div></article></div></section>
        <section v-else class="view-section"><div class="page-heading"><div><span class="eyebrow">PREFERENCES</span><h1>Ruangmu, atur sesukamu.</h1><p class="muted">Pengaturan tampilan dan pengalaman Dompetku.</p></div></div><article class="panel settings-panel"><div><strong>Personal account</strong><p class="muted">Andi Ramadhan</p></div><button class="select-button">Edit profil <ArrowUpRight :size="15"/></button><hr/><div><strong>Appearance</strong><p class="muted">Dark mode aktif untuk fokus yang lebih nyaman.</p></div><div class="theme-pill"><Sparkles :size="16"/>Dark neon</div></article></section>
      </div>
    </main>
    <Transition name="modal"><div v-if="showModal" class="modal-backdrop" @click.self="showModal = false"><form class="modal" @submit.prevent="addTransaction"><button type="button" class="close-modal" @click="showModal = false"><X/></button><span class="eyebrow">NEW ACTIVITY</span><h2>Tambah transaksi</h2><p class="muted">Catat sekali, biar pikiran lebih ringan.</p><label>Nama transaksi<input v-model="form.name" placeholder="Contoh: Freelance project" required/></label><label>Kategori<input v-model="form.category" placeholder="Contoh: Lainnya"/></label><label>Jumlah<input v-model="form.amount" type="number" min="1" placeholder="0" required/></label><div class="type-toggle"><button type="button" :class="{ active: form.type === 'income' }" @click="form.type = 'income'">Pemasukan</button><button type="button" :class="{ active: form.type === 'expense' }" @click="form.type = 'expense'">Pengeluaran</button></div><button class="primary-button full" type="submit"><Plus :size="16"/>Simpan transaksi</button></form></div></Transition>
  </div>
    <Transition name="modal"><div v-if="showGoalModal" class="modal-backdrop" @click.self="showGoalModal = false"><form class="modal goal-modal" @submit.prevent="saveGoal"><button type="button" class="close-modal" @click="showGoalModal = false"><X/></button><span class="eyebrow">NEW GOAL</span><h2>Buat target finansial</h2><p class="muted">Tentukan tujuanmu, lalu pantau progresnya.</p><label>Nama target<input v-model="goalDraft.name" placeholder="Contoh: Dana darurat" required/></label><label>Nominal target<input v-model="goalDraft.target" type="number" min="1" placeholder="10000000" required/></label><label>Dana terkumpul<input v-model="goalDraft.saved" type="number" min="0" placeholder="0"/></label><label>Deadline<input v-model="goalDraft.deadline" placeholder="Contoh: Desember 2026"/></label><button class="primary-button full" type="submit"><Plus :size="16"/>Simpan target</button></form></div></Transition>
    <Transition name="modal"><div v-if="showFundModal" class="modal-backdrop" @click.self="showFundModal = false"><form class="modal fund-modal" @submit.prevent="saveGoalFunds"><button type="button" class="close-modal" @click="showFundModal = false"><X/></button><span class="eyebrow">GOAL FUNDING</span><h2>Tambah dana</h2><p class="muted">Alokasikan dana ke <strong>{{ selectedGoal?.name }}</strong>.</p><label>Jumlah dana<input v-model="fundAmount" type="number" min="1" placeholder="0" required/></label><button class="primary-button full" type="submit"><Plus :size="16"/>Tambahkan dana</button></form></div></Transition>
</template>

<script lang="ts">
import { Plane as PlaneIcon } from "lucide-vue-next";
export default { components: { PlaneIcon } };
</script>
