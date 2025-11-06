<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { browser } from '$app/environment';
  import { supabase } from '$lib/supabaseClient';

  let adminSession = null;
  let loading = false;
  let error = '';
  let success = '';

  let votingStart = '';
  let votingEnd = '';
  let currentSettings = null;

  onMount(async () => {
    if (browser) {
      const session = localStorage.getItem('admin_session');
      if (!session) {
        goto('/admin');
        return;
      }
      adminSession = JSON.parse(session);
    }

    await loadCurrentSettings();
  });

  async function loadCurrentSettings() {
    loading = true;
    try {
      const { data, error: dbError } = await supabase
        .from('voting_settings')
        .select('*')
        .eq('is_active', true)
        .order('created_at', { ascending: false })
        .limit(1);

      console.log('Loaded settings:', data);

      if (data && data.length > 0) {
        currentSettings = data[0];
        // Convert to local datetime format for input
        const startDate = new Date(data[0].voting_start);
        const endDate = new Date(data[0].voting_end);
        
        // Subtract timezone offset to get local time
        startDate.setMinutes(startDate.getMinutes() - startDate.getTimezoneOffset());
        endDate.setMinutes(endDate.getMinutes() - endDate.getTimezoneOffset());
        
        votingStart = startDate.toISOString().slice(0, 16);
        votingEnd = endDate.toISOString().slice(0, 16);
      }
    } catch (err) {
      console.error('Error loading settings:', err);
    }
    loading = false;
  }

  async function saveVotingTime() {
    if (!votingStart || !votingEnd) {
      error = 'Sila isi tarikh dan masa mula dan tamat';
      return;
    }

    const startDate = new Date(votingStart);
    const endDate = new Date(votingEnd);

    if (endDate <= startDate) {
      error = 'Masa tamat mestilah selepas masa mula';
      return;
    }

    loading = true;
    error = '';
    success = '';

    try {
      console.log('Saving dates:', {
        start: startDate.toISOString(),
        end: endDate.toISOString()
      });

      // Deactivate all old settings
      await supabase
        .from('voting_settings')
        .update({ is_active: false })
        .neq('id', '00000000-0000-0000-0000-000000000000');

      // Insert new settings
      const { data: newData, error: insertError } = await supabase
        .from('voting_settings')
        .insert([{
          voting_start: startDate.toISOString(),
          voting_end: endDate.toISOString(),
          is_active: true,
          created_by: adminSession?.email
        }])
        .select();

      if (insertError) {
        console.error('Insert error:', insertError);
        throw insertError;
      }

      console.log('Saved successfully:', newData);

      success = 'Masa pengundian berjaya ditetapkan!';
      
      // Reload settings
      await loadCurrentSettings();
      
      loading = false;

      // Auto dismiss success message after 3 seconds
      setTimeout(() => {
        success = '';
      }, 3000);

    } catch (err) {
      console.error('Save error:', err);
      error = 'Gagal simpan: ' + err.message;
      loading = false;
    }
  }

  function getVotingStatus() {
    if (!currentSettings) return 'none';
    
    const now = new Date();
    const start = new Date(currentSettings.voting_start);
    const end = new Date(currentSettings.voting_end);

    if (now < start) return 'upcoming';
    if (now > end) return 'closed';
    return 'active';
  }

  $: votingStatus = getVotingStatus();
</script>

<div class="min-h-screen bg-gradient-to-br from-gray-50 to-orange-100">
  <!-- Header -->
  <div class="bg-gradient-to-r from-orange-600 to-red-600 shadow-lg">
    <div class="max-w-7xl mx-auto px-4 py-6">
      <div class="flex items-center gap-4">
        <a href="/admin/dashboard" class="text-white hover:text-orange-200 transition">
          <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 19l-7-7m0 0l7-7m-7 7h18"/>
          </svg>
        </a>
        <div>
          <h1 class="text-2xl font-bold text-white">Tetapkan Masa Pengundian</h1>
          <p class="text-orange-200 text-sm">Set tarikh dan masa mula/tamat pengundian</p>
        </div>
      </div>
    </div>
  </div>

  <div class="max-w-4xl mx-auto px-4 py-8">
    {#if error}
      <div class="bg-red-100 border-l-4 border-red-500 text-red-700 px-4 py-3 rounded-lg mb-6">
        <strong>⚠️ Ralat:</strong> {error}
      </div>
    {/if}

    {#if success}
      <div class="bg-green-100 border-l-4 border-green-500 text-green-700 px-4 py-3 rounded-lg mb-6">
        <strong>✅ Berjaya!</strong> {success}
      </div>
    {/if}

    <!-- Current Status -->
    {#if currentSettings}
      <div class="bg-white rounded-xl shadow-lg p-6 mb-6">
        <h2 class="text-xl font-bold text-gray-800 mb-4">Status Semasa</h2>
        
        <div class="grid md:grid-cols-3 gap-4">
          <div class="text-center p-4 bg-gray-50 rounded-lg">
            <p class="text-sm text-gray-600 mb-1">Masa Mula</p>
            <p class="font-bold text-gray-800">
              {new Date(currentSettings.voting_start).toLocaleString('ms-MY', { 
                dateStyle: 'medium', 
                timeStyle: 'short' 
              })}
            </p>
          </div>

          <div class="text-center p-4 bg-gray-50 rounded-lg">
            <p class="text-sm text-gray-600 mb-1">Masa Tamat</p>
            <p class="font-bold text-gray-800">
              {new Date(currentSettings.voting_end).toLocaleString('ms-MY', { 
                dateStyle: 'medium', 
                timeStyle: 'short' 
              })}
            </p>
          </div>

          <div class="text-center p-4 rounded-lg {
            votingStatus === 'active' ? 'bg-green-100' :
            votingStatus === 'upcoming' ? 'bg-yellow-100' :
            'bg-red-100'
          }">
            <p class="text-sm text-gray-600 mb-1">Status</p>
            <p class="font-bold text-lg {
              votingStatus === 'active' ? 'text-green-700' :
              votingStatus === 'upcoming' ? 'text-yellow-700' :
              'text-red-700'
            }">
              {votingStatus === 'active' ? '🟢 Aktif' :
               votingStatus === 'upcoming' ? '🟡 Akan Datang' :
               '🔴 Ditutup'}
            </p>
          </div>
        </div>
      </div>
    {/if}

    <!-- Set New Time -->
    <div class="bg-white rounded-xl shadow-lg p-8">
      <h2 class="text-2xl font-bold text-gray-800 mb-6">
        {currentSettings ? 'Kemaskini' : 'Tetapkan'} Masa Pengundian
      </h2>

      <form on:submit|preventDefault={saveVotingTime} class="space-y-6">
        <!-- Voting Start -->
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-2">
            Tarikh & Masa Mula Pengundian <span class="text-red-500">*</span>
          </label>
          <input
            type="datetime-local"
            bind:value={votingStart}
            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-orange-500 focus:border-transparent"
            required
          />
          <p class="text-xs text-gray-500 mt-1">
            Pengundian akan dibuka pada masa ini
          </p>
        </div>

        <!-- Voting End -->
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-2">
            Tarikh & Masa Tamat Pengundian <span class="text-red-500">*</span>
          </label>
          <input
            type="datetime-local"
            bind:value={votingEnd}
            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-orange-500 focus:border-transparent"
            required
          />
          <p class="text-xs text-gray-500 mt-1">
            Pengundian akan ditutup pada masa ini
          </p>
        </div>

        <!-- Info Box -->
        <div class="bg-blue-50 border-l-4 border-blue-500 p-4 rounded">
          <div class="flex items-start gap-3">
            <svg class="w-5 h-5 text-blue-500 flex-shrink-0 mt-0.5" fill="currentColor" viewBox="0 0 20 20">
              <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z" clip-rule="evenodd"/>
            </svg>
            <div class="text-sm text-blue-700">
              <p class="font-semibold mb-1">Nota Penting:</p>
              <ul class="space-y-1">
                <li>• Pengundi hanya boleh mengundi dalam masa yang ditetapkan</li>
                <li>• System akan automatik check masa sebelum allow voting</li>
                <li>• Anda boleh update masa ini bila-bila masa</li>
              </ul>
            </div>
          </div>
        </div>

        <!-- Buttons -->
        <div class="flex gap-4">
          <button
            type="submit"
            disabled={loading}
            class="flex-1 bg-gradient-to-r from-orange-600 to-red-600 text-white py-4 rounded-lg font-semibold hover:from-orange-700 hover:to-red-700 disabled:opacity-50 transition shadow-lg"
          >
            {loading ? '⏳ Menyimpan...' : '🕐 Tetapkan Masa'}
          </button>
          <a
            href="/admin/dashboard"
            class="px-8 py-4 border-2 border-gray-300 rounded-lg font-semibold hover:bg-gray-50 transition"
          >
            Batal
          </a>
        </div>
      </form>
    </div>

    <!-- Quick Links -->
    <div class="mt-6 text-center">
      <a href="/vote" class="text-orange-600 hover:text-orange-700 font-medium hover:underline">
        👉 Preview Page Pengundian
      </a>
    </div>
  </div>
</div>