<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let step = 'verify'; // verify, vote, success, closed, upcoming
  let loading = true;
  let error = '';
  
  let voterEmail = '';
  let voter = null;
  let votingSettings = null;
  let candidates = {
    president: [],
    vicePresident: [],
    exco: []
  };

  let selectedVotes = {
    president: null,
    vicePresident: null,
    exco: []
  };

  onMount(async () => {
    await checkVotingStatus();
  });

  async function checkVotingStatus() {
    loading = true;

    try {
      // Get voting settings
      const { data: settings } = await supabase
        .from('voting_settings')
        .select('*')
        .eq('is_active', true)
        .order('created_at', { ascending: false })
        .limit(1)
        .single();

      if (!settings) {
        step = 'closed';
        loading = false;
        return;
      }

      votingSettings = settings;
      const now = new Date();
      const start = new Date(settings.voting_start);
      const end = new Date(settings.voting_end);

      if (now < start) {
        step = 'upcoming';
        loading = false;
        return;
      }

      if (now > end) {
        step = 'closed';
        loading = false;
        return;
      }

      // Load candidates
      await loadCandidates();
      step = 'verify';
      loading = false;

    } catch (err) {
      error = 'Gagal load data: ' + err.message;
      loading = false;
    }
  }

  async function loadCandidates() {
    const { data } = await supabase
      .from('voting_candidates')
      .select('*')
      .order('display_order');

    if (data) {
      candidates.president = data.filter(c => c.position === 'president');
      candidates.vicePresident = data.filter(c => c.position === 'vice_president');
      candidates.exco = data.filter(c => c.position === 'exco');
    }
  }

  async function verifyVoter() {
    if (!voterEmail) {
      error = 'Sila masukkan email';
      return;
    }

    loading = true;
    error = '';

    try {
      const { data, error: dbError } = await supabase
        .from('voters')
        .select('*')
        .eq('email', voterEmail);

      if (dbError || !data || data.length === 0) {
        error = 'Email tidak dijumpai. Sila daftar terlebih dahulu.';
        loading = false;
        return;
      }

      voter = data[0];

      if (voter.has_voted) {
        error = 'Anda sudah mengundi! Setiap pengundi hanya boleh mengundi sekali.';
        loading = false;
        return;
      }

      step = 'vote';
      loading = false;

    } catch (err) {
      error = 'Ralat: ' + err.message;
      loading = false;
    }
  }

  function toggleExcoVote(candidateId) {
    const index = selectedVotes.exco.indexOf(candidateId);
    if (index > -1) {
      selectedVotes.exco = selectedVotes.exco.filter(id => id !== candidateId);
    } else {
      if (selectedVotes.exco.length < 7) {
        selectedVotes.exco = [...selectedVotes.exco, candidateId];
      } else {
        error = 'Anda hanya boleh pilih maksimum 7 calon Exco';
        setTimeout(() => error = '', 3000);
      }
    }
  }

  async function submitVote() {
    // Validation
    if (!selectedVotes.president) {
      error = 'Sila pilih calon Presiden';
      return;
    }
    if (!selectedVotes.vicePresident) {
      error = 'Sila pilih calon Timbalan/Naib Presiden';
      return;
    }
    if (selectedVotes.exco.length === 0) {
      error = 'Sila pilih sekurang-kurangnya 1 calon Exco';
      return;
    }

    loading = true;
    error = '';

    try {
      // Insert vote
      const { error: voteError } = await supabase
        .from('votes')
        .insert([{
          voter_id: voter.id,
          president_vote: selectedVotes.president,
          vice_president_vote: selectedVotes.vicePresident,
          exco_votes: selectedVotes.exco
        }]);

      if (voteError) throw voteError;

      // Update voter status
      const { error: updateError } = await supabase
        .from('voters')
        .update({ 
          has_voted: true,
          voted_at: new Date().toISOString()
        })
        .eq('id', voter.id);

      if (updateError) throw updateError;

      step = 'success';
      loading = false;

    } catch (err) {
      error = 'Gagal hantar undi: ' + err.message;
      loading = false;
    }
  }
</script>

<div class="min-h-screen bg-gradient-to-br from-green-50 to-teal-100 py-12 px-4">
  <div class="max-w-5xl mx-auto">
    <!-- Header -->
    <div class="text-center mb-8">
      <h1 class="text-4xl font-bold text-teal-900 mb-2">
        🗳️ Undi Sekarang
      </h1>
      <p class="text-gray-600">Sistem Pengundian Persatuan</p>
    </div>

    {#if loading && step !== 'vote'}
      <!-- Loading -->
      <div class="bg-white rounded-xl shadow-lg p-12 text-center">
        <div class="animate-spin rounded-full h-16 w-16 border-b-2 border-teal-600 mx-auto mb-4"></div>
        <p class="text-gray-600">Memuatkan...</p>
      </div>

    {:else if step === 'upcoming'}
      <!-- Not Started Yet -->
      <div class="bg-white rounded-xl shadow-lg p-12 text-center">
        <div class="text-6xl mb-4">⏰</div>
        <h2 class="text-3xl font-bold text-yellow-600 mb-4">Pengundian Belum Dibuka</h2>
        <p class="text-gray-600 text-lg mb-6">
          Pengundian akan dibuka pada:
        </p>
        <div class="bg-yellow-50 border-2 border-yellow-300 rounded-lg p-6 inline-block">
          <p class="text-2xl font-bold text-yellow-800">
            {new Date(votingSettings.voting_start).toLocaleString('ms-MY', { 
              dateStyle: 'full', 
              timeStyle: 'short' 
            })}
          </p>
        </div>
        <p class="text-sm text-gray-500 mt-6">
          Sila kembali pada masa yang ditetapkan
        </p>
        <a href="/" class="inline-block mt-6 bg-teal-600 text-white px-6 py-3 rounded-lg font-semibold hover:bg-teal-700">
          Kembali ke Utama
        </a>
      </div>

    {:else if step === 'closed'}
      <!-- Closed -->
      <div class="bg-white rounded-xl shadow-lg p-12 text-center">
        <div class="text-6xl mb-4">🔒</div>
        <h2 class="text-3xl font-bold text-red-600 mb-4">Pengundian Telah Ditutup</h2>
        <p class="text-gray-600 text-lg mb-6">
          Terima kasih atas penyertaan anda.
        </p>
        {#if votingSettings}
          <div class="bg-red-50 border-2 border-red-300 rounded-lg p-4 inline-block">
            <p class="text-sm text-gray-600 mb-1">Pengundian ditutup pada:</p>
            <p class="text-lg font-bold text-red-800">
              {new Date(votingSettings.voting_end).toLocaleString('ms-MY', { 
                dateStyle: 'full', 
                timeStyle: 'short' 
              })}
            </p>
          </div>
        {/if}
        <div class="mt-8 space-x-4">
          <a href="/results" class="inline-block bg-purple-600 text-white px-6 py-3 rounded-lg font-semibold hover:bg-purple-700">
            📊 Lihat Keputusan
          </a>
          <a href="/" class="inline-block bg-gray-200 text-gray-700 px-6 py-3 rounded-lg font-semibold hover:bg-gray-300">
            Kembali ke Utama
          </a>
        </div>
      </div>

    {:else if step === 'verify'}
      <!-- Email Verification -->
      <div class="bg-white rounded-xl shadow-lg p-8">
        <div class="max-w-md mx-auto">
          <div class="text-center mb-6">
            <div class="bg-teal-100 rounded-full w-20 h-20 flex items-center justify-center mx-auto mb-4">
              <span class="text-4xl">✉️</span>
            </div>
            <h2 class="text-2xl font-bold text-gray-800 mb-2">Verify Email Anda</h2>
            <p class="text-gray-600">Masukkan email yang didaftarkan untuk mengundi</p>
          </div>

          {#if error}
            <div class="bg-red-100 border-l-4 border-red-500 text-red-700 px-4 py-3 rounded mb-4">
              {error}
            </div>
          {/if}

          {#if votingSettings}
            <div class="bg-green-50 border-l-4 border-green-500 p-4 rounded mb-6">
              <p class="text-sm text-green-700">
                <strong>🟢 Pengundian Aktif</strong><br/>
                Tamat: {new Date(votingSettings.voting_end).toLocaleString('ms-MY', { 
                  dateStyle: 'medium', 
                  timeStyle: 'short' 
                })}
              </p>
            </div>
          {/if}

          <form on:submit|preventDefault={verifyVoter} class="space-y-4">
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-2">Email</label>
              <input
                type="email"
                bind:value={voterEmail}
                placeholder="email@example.com"
                class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-teal-500 focus:border-transparent"
                required
              />
            </div>

            <button
              type="submit"
              disabled={loading}
              class="w-full bg-teal-600 text-white py-3 rounded-lg font-semibold hover:bg-teal-700 disabled:opacity-50 transition"
            >
              {loading ? 'Memeriksa...' : 'Teruskan'}
            </button>
          </form>

          <p class="text-center text-sm text-gray-500 mt-4">
            Belum daftar? <a href="/register" class="text-teal-600 hover:underline font-medium">Daftar Sekarang</a>
          </p>
        </div>
      </div>

    {:else if step === 'vote'}
      <!-- Voting Form -->
      <div class="bg-white rounded-xl shadow-lg p-8">
        <div class="mb-6 pb-6 border-b">
          <h2 class="text-2xl font-bold text-gray-800">Selamat Mengundi, {voter.full_name}!</h2>
          <p class="text-sm text-gray-600 mt-1">{voter.email}</p>
        </div>

        {#if error}
          <div class="bg-red-100 border-l-4 border-red-500 text-red-700 px-4 py-3 rounded mb-6">
            {error}
          </div>
        {/if}

        <div class="space-y-8">
          <!-- President -->
          <div>
            <h3 class="text-xl font-bold text-gray-800 mb-4 flex items-center">
              <span class="bg-purple-600 text-white rounded-full w-8 h-8 flex items-center justify-center mr-3 text-sm">1</span>
              Presiden
            </h3>
            <div class="space-y-3">
              {#each candidates.president as candidate}
                <button
                  on:click={() => selectedVotes.president = candidate.id}
                  class="w-full text-left p-4 border-2 rounded-lg transition {
                    selectedVotes.president === candidate.id 
                      ? 'border-purple-600 bg-purple-50' 
                      : 'border-gray-200 hover:border-purple-300'
                  }"
                >
                  <div class="flex items-center justify-between">
                    <div class="flex-1">
                      <p class="font-semibold text-gray-800">{candidate.candidate_name}</p>
                    </div>
                    {#if selectedVotes.president === candidate.id}
                      <div class="text-purple-600 text-2xl">✓</div>
                    {/if}
                  </div>
                </button>
              {/each}
            </div>
          </div>

          <!-- Vice President -->
          <div>
            <h3 class="text-xl font-bold text-gray-800 mb-4 flex items-center">
              <span class="bg-blue-600 text-white rounded-full w-8 h-8 flex items-center justify-center mr-3 text-sm">1</span>
              Timbalan Presiden / Naib Presiden
            </h3>
            <div class="space-y-3">
              {#each candidates.vicePresident as candidate}
                <button
                  on:click={() => selectedVotes.vicePresident = candidate.id}
                  class="w-full text-left p-4 border-2 rounded-lg transition {
                    selectedVotes.vicePresident === candidate.id 
                      ? 'border-blue-600 bg-blue-50' 
                      : 'border-gray-200 hover:border-blue-300'
                  }"
                >
                  <div class="flex items-center justify-between">
                    <div class="flex-1">
                      <p class="font-semibold text-gray-800">{candidate.candidate_name}</p>
                    </div>
                    {#if selectedVotes.vicePresident === candidate.id}
                      <div class="text-blue-600 text-2xl">✓</div>
                    {/if}
                  </div>
                </button>
              {/each}
            </div>
          </div>

          <!-- Exco -->
          <div>
            <h3 class="text-xl font-bold text-gray-800 mb-2 flex items-center">
              <span class="bg-green-600 text-white rounded-full w-8 h-8 flex items-center justify-center mr-3 text-sm">7</span>
              Exco
            </h3>
            <p class="text-sm text-gray-600 mb-4">Pilih maksimum 7 calon. Dipilih: {selectedVotes.exco.length}/7</p>
            
            <div class="grid md:grid-cols-2 gap-3">
              {#each candidates.exco as candidate}
                <button
                  on:click={() => toggleExcoVote(candidate.id)}
                  disabled={!selectedVotes.exco.includes(candidate.id) && selectedVotes.exco.length >= 7}
                  class="text-left p-4 border-2 rounded-lg transition disabled:opacity-50 disabled:cursor-not-allowed {
  selectedVotes.exco.includes(candidate.id) ? 'bg-green-100 border-green-500' : 'bg-white border-gray-300'
}"