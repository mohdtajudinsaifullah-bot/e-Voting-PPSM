<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { browser } from '$app/environment';
  import { supabase } from '$lib/supabaseClient';

  let adminSession = null;
  let stats = {
    totalVoters: 0,
    totalNominations: 0,
    totalVotingCandidates: 0,
    votedCount: 0
  };
  let loading = true;

  onMount(async () => {
    if (browser) {
      const session = localStorage.getItem('admin_session');
      if (!session) {
        goto('/admin');
        return;
      }
      adminSession = JSON.parse(session);
    }

    await loadStats();
  });

  async function loadStats() {
    loading = true;
    try {
      const { count: votersCount } = await supabase
        .from('voters')
        .select('*', { count: 'exact', head: true });
      stats.totalVoters = votersCount || 0;

      const { count: nomCount } = await supabase
        .from('nominations')
        .select('*', { count: 'exact', head: true });
      stats.totalNominations = nomCount || 0;

      const { count: candidatesCount } = await supabase
        .from('voting_candidates')
        .select('*', { count: 'exact', head: true });
      stats.totalVotingCandidates = candidatesCount || 0;

      const { count: votedCount } = await supabase
        .from('voters')
        .select('*', { count: 'exact', head: true })
        .eq('has_voted', true);
      stats.votedCount = votedCount || 0;

    } catch (err) {
      console.error('Error loading stats:', err);
    }
    loading = false;
  }

  function logout() {
    if (browser) {
      localStorage.removeItem('admin_session');
    }
    goto('/admin');
  }
</script>

<div class="min-h-screen bg-gray-50">
  <!-- Top Bar -->
  <div class="bg-white border-b border-gray-200 shadow-sm">
    <div class="max-w-7xl mx-auto px-6 py-4">
      <div class="flex items-center justify-between">
        <div>
          <h1 class="text-2xl font-bold text-gray-800">Admin Dashboard</h1>
          <p class="text-sm text-gray-600 mt-1">Selamat datang, {adminSession?.name || 'Admin'}</p>
        </div>
        <button
          on:click={logout}
          class="px-5 py-2.5 bg-red-500 hover:bg-red-600 text-white rounded-lg font-medium transition shadow"
        >
          🚪 Log Keluar
        </button>
      </div>
    </div>
  </div>

  <!-- Main Content -->
  <div class="max-w-7xl mx-auto px-6 py-8">
    {#if loading}
      <div class="flex items-center justify-center py-32">
        <div class="text-center">
          <div class="inline-block animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600 mb-4"></div>
          <p class="text-gray-600 font-medium">Memuatkan data...</p>
        </div>
      </div>
    {:else}
      <!-- Stats Grid -->
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
        <!-- Card 1 -->
        <div class="bg-white rounded-xl shadow-md p-6 border-l-4 border-blue-500">
          <div class="flex items-center justify-between">
            <div>
              <p class="text-sm text-gray-600 font-medium mb-1">Jumlah Pengundi</p>
              <p class="text-3xl font-bold text-gray-800">{stats.totalVoters}</p>
            </div>
            <div class="text-5xl">👥</div>
          </div>
        </div>

        <!-- Card 2 -->
        <div class="bg-white rounded-xl shadow-md p-6 border-l-4 border-purple-500">
          <div class="flex items-center justify-between">
            <div>
              <p class="text-sm text-gray-600 font-medium mb-1">Pencalonan</p>
              <p class="text-3xl font-bold text-gray-800">{stats.totalNominations}</p>
            </div>
            <div class="text-5xl">📝</div>
          </div>
        </div>

        <!-- Card 3 -->
        <div class="bg-white rounded-xl shadow-md p-6 border-l-4 border-green-500">
          <div class="flex items-center justify-between">
            <div>
              <p class="text-sm text-gray-600 font-medium mb-1">Calon Undi</p>
              <p class="text-3xl font-bold text-gray-800">{stats.totalVotingCandidates}</p>
            </div>
            <div class="text-5xl">🎯</div>
          </div>
        </div>

        <!-- Card 4 -->
        <div class="bg-white rounded-xl shadow-md p-6 border-l-4 border-orange-500">
          <div class="flex items-center justify-between">
            <div>
              <p class="text-sm text-gray-600 font-medium mb-1">Sudah Mengundi</p>
              <p class="text-3xl font-bold text-gray-800">{stats.votedCount}</p>
            </div>
            <div class="text-5xl">✅</div>
          </div>
        </div>
      </div>

      <!-- Menu Card -->
      <div class="bg-white rounded-xl shadow-lg p-8">
        <h2 class="text-2xl font-bold text-gray-800 mb-6 text-center">Modul Pengurusan</h2>
        
        <div class="max-w-3xl mx-auto space-y-4">
          <!-- Button 1 -->
          <a
            href="/admin/voting-time"
            class="block w-full bg-gradient-to-r from-orange-500 to-red-500 hover:from-orange-600 hover:to-red-600 text-white px-6 py-4 rounded-xl font-semibold text-center transition shadow-lg hover:shadow-xl transform hover:-translate-y-1"
          >
            🕐 Tetapkan Masa Pengundian
          </a>

          <!-- Button 2 -->
          <a
            href="/admin/voters"
            class="block w-full bg-gradient-to-r from-blue-500 to-indigo-500 hover:from-blue-600 hover:to-indigo-600 text-white px-6 py-4 rounded-xl font-semibold text-center transition shadow-lg hover:shadow-xl transform hover:-translate-y-1"
          >
            👥 Kemaskini Maklumat Pengundi
          </a>

          <!-- Button 3 -->
          <a
            href="/admin/candidates"
            class="block w-full bg-gradient-to-r from-purple-500 to-pink-500 hover:from-purple-600 hover:to-pink-600 text-white px-6 py-4 rounded-xl font-semibold text-center transition shadow-lg hover:shadow-xl transform hover:-translate-y-1"
          >
            🎯 Masukkan Calon Undi
          </a>

          <!-- Divider -->
          <div class="relative py-3">
            <div class="absolute inset-0 flex items-center">
              <div class="w-full border-t border-gray-300"></div>
            </div>
            <div class="relative flex justify-center">
              <span class="px-4 bg-white text-sm text-gray-500">atau</span>
            </div>
          </div>

          <!-- Button 4 -->
          <a
            href="/"
            class="block w-full bg-gray-100 hover:bg-gray-200 text-gray-700 px-6 py-4 rounded-xl font-semibold text-center transition shadow hover:shadow-md"
          >
            🏠 Kembali ke Halaman Utama
          </a>
        </div>
      </div>

      <!-- Quick Links -->
      <div class="mt-8 text-center">
        <div class="inline-flex flex-wrap items-center justify-center gap-4 text-sm">
          <a href="/candidates" class="text-blue-600 hover:text-blue-700 font-medium hover:underline">Senarai Pencalonan</a>
          <span class="text-gray-400">•</span>
          <a href="/vote" class="text-blue-600 hover:text-blue-700 font-medium hover:underline">Preview Pengundian</a>
          <span class="text-gray-400">•</span>
          <a href="/register" class="text-blue-600 hover:text-blue-700 font-medium hover:underline">Daftar Pengundi</a>
        </div>
      </div>
    {/if}
  </div>
</div>

<style>
  @keyframes spin {
    to { transform: rotate(360deg); }
  }
  
  .animate-spin {
    animation: spin 1s linear infinite;
  }
</style>