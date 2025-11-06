<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let voterEmail = '';
  let voterName = '';
  let loading = true;
  let error = '';
  let success = '';
  let voting = false;

  // Voting status
  let votingActive = false;
  let votingMessage = '';
  let votingSettings = null;

  // Voter status
  let isRegistered = false;
  let hasVoted = false;

  // Positions and candidates
  let enabledPositions = [];
  let votes = {}; // Store selected candidates

  onMount(() => {
    checkVotingStatus();
  });

  async function checkVotingStatus() {
    loading = true;
    try {
      // Check voting settings
      const { data: settings } = await supabase
        .from('voting_settings')
        .select('*')
        .order('created_at', { ascending: false })
        .limit(1)
        .single();

      if (settings) {
        votingSettings = settings;
        const now = new Date();
        const start = new Date(settings.start_date);
        const end = new Date(settings.end_date);

        if (!settings.is_active) {
          votingActive = false;
          votingMessage = 'Pengundian tidak aktif';
        } else if (now < start) {
          votingActive = false;
          votingMessage = `Pengundian akan bermula pada ${formatDateTime(settings.start_date)}`;
        } else if (now > end) {
          votingActive = false;
          votingMessage = 'Tempoh pengundian telah tamat';
        } else {
          votingActive = true;
          votingMessage = `Pengundian aktif sehingga ${formatDateTime(settings.end_date)}`;
        }
      } else {
        votingActive = false;
        votingMessage = 'Tiada tempoh pengundian ditetapkan';
      }

    } catch (err) {
      console.error('Error checking voting status:', err);
      votingActive = false;
      votingMessage = 'Ralat memeriksa status pengundian';
    } finally {
      loading = false;
    }
  }

  async function handleLogin() {
    if (!voterEmail.trim()) {
      error = 'Sila masukkan email anda';
      return;
    }

    loading = true;
    error = '';

    try {
      // Check if voter is registered
      const { data: voter } = await supabase
        .from('voters')
        .select('*')
        .eq('email', voterEmail.toLowerCase().trim())
        .single();

      if (!voter) {
        error = 'Email tidak berdaftar. Sila hubungi pentadbir.';
        loading = false;
        return;
      }

      voterName = voter.full_name;
      isRegistered = true;

      // Check if already voted
      const { data: existingVotes } = await supabase
        .from('votes')
        .select('position')
        .eq('voter_email', voterEmail.toLowerCase().trim());

      if (existingVotes && existingVotes.length > 0) {
        hasVoted = true;
        loading = false;
        return;
      }

      // Load enabled positions and candidates
      await loadVotingData();

    } catch (err) {
      console.error('Login error:', err);
      error = 'Ralat log masuk: ' + err.message;
    } finally {
      loading = false;
    }
  }

  async function loadVotingData() {
    try {
      // Get enabled positions
      const { data: positions } = await supabase
        .from('position_settings')
        .select('*')
        .eq('is_enabled', true)
        .order('position');

      if (!positions || positions.length === 0) {
        error = 'Tiada jawatan dibuka untuk pengundian';
        return;
      }

      // Get candidates for enabled positions
      const positionNames = positions.map(p => p.position);
      const { data: candidates } = await supabase
        .from('voting_candidates')
        .select('*')
        .in('position', positionNames)
        .eq('is_enabled', true)
        .order('position')
        .order('display_order');

      // Group candidates by position
      enabledPositions = positions.map(pos => ({
        ...pos,
        candidates: candidates?.filter(c => c.position === pos.position) || []
      }));

      // Initialize votes object
      enabledPositions.forEach(pos => {
        votes[pos.position] = null;
      });

    } catch (err) {
      console.error('Error loading voting data:', err);
      error = 'Ralat memuatkan data: ' + err.message;
    }
  }

  function selectCandidate(position, candidateName) {
    votes[position] = candidateName;
  }

  async function handleSubmitVotes() {
    // Validate at least one vote
    const selectedVotes = Object.entries(votes).filter(([pos, cand]) => cand !== null);
    
    if (selectedVotes.length === 0) {
      error = 'Sila pilih sekurang-kurangnya satu calon';
      return;
    }

    if (!confirm(`Anda telah memilih ${selectedVotes.length} jawatan. Adakah anda pasti? Undi tidak boleh diubah.`)) {
      return;
    }

    voting = true;
    error = '';

    try {
      // Prepare votes data
      const votesData = selectedVotes.map(([position, candidateName]) => ({
        voter_email: voterEmail.toLowerCase().trim(),
        voter_name: voterName,
        position: position,
        candidate_name: candidateName
      }));

      // Insert votes
      const { error: dbError } = await supabase
        .from('votes')
        .insert(votesData);

      if (dbError) throw dbError;

      success = 'Undi anda telah berjaya direkodkan! Terima kasih.';
      hasVoted = true;

    } catch (err) {
      console.error('Error submitting votes:', err);
      if (err.message.includes('duplicate')) {
        error = 'Anda sudah mengundi untuk jawatan ini';
        hasVoted = true;
      } else {
        error = 'Gagal menghantar undi: ' + err.message;
      }
    } finally {
      voting = false;
    }
  }

  function formatDateTime(dateStr) {
    const date = new Date(dateStr);
    return date.toLocaleString('ms-MY', {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
      hour: '2-digit',
      minute: '2-digit',
      hour12: true
    });
  }

  function getPositionIcon(position) {
    if (position.includes('Presiden')) return '👑';
    if (position.includes('Exco')) return '💼';
    if (position.includes('Pemeriksa')) return '📊';
    return '🎯';
  }
</script>

<svelte:head>
  <title>Pengundian - e-VOTING PPSM</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    body {
      overflow-y: auto !important;
    }
  </style>
</svelte:head>

<!-- NO SIDEBAR - Full Width Layout -->
<div style="min-height: 100vh; background: #f9fafb;">
  
  <!-- Header -->
  <div style="background: linear-gradient(135deg, #14b8a6 0%, #06b6d4 50%, #3b82f6 100%); padding: 2rem 1rem; color: white; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
    <div style="max-width: 1200px; margin: 0 auto;">
      <div style="text-align: center;">
        <h1 style="font-size: 3rem; font-weight: 700; text-shadow: 2px 2px 4px rgba(0,0,0,0.2); letter-spacing: 0.05em; margin-bottom: 0.5rem;">
          e-VOTING PPSM
        </h1>
        <p style="font-size: 1.125rem; opacity: 0.95;">Sistem Pengundian Persatuan</p>
      </div>
    </div>
  </div>

  <!-- Main Content - Full Width -->
  <div style="max-width: 1200px; margin: 0 auto; padding: 2rem;">
    
    <div style="margin-bottom: 2rem;">
      <h2 style="font-size: 2rem; font-weight: 700; color: #111827; display: flex; align-items: center; gap: 0.75rem; margin-bottom: 0.5rem;">
        🗳️ Undi Sekarang
      </h2>
    </div>

    {#if loading}
      <div style="text-align: center; padding: 3rem; color: #6b7280;">
        ⏳ Memuatkan...
      </div>
    {:else if !votingActive}
      <!-- Voting Not Active -->
      <div style="background: white; border-radius: 16px; padding: 3rem; text-align: center; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
        <div style="font-size: 4rem; margin-bottom: 1rem;">⏸️</div>
        <h3 style="font-size: 1.5rem; font-weight: 700; color: #6b7280; margin-bottom: 0.5rem;">
          {votingMessage}
        </h3>
        {#if votingSettings}
          <p style="color: #9ca3af; margin-top: 1rem;">
            Tempoh: {formatDateTime(votingSettings.start_date)} - {formatDateTime(votingSettings.end_date)}
          </p>
        {/if}
        <a href="/" style="display: inline-block; margin-top: 2rem; color: #3b82f6; text-decoration: none; font-weight: 600;">
          ← Kembali ke Halaman Utama
        </a>
      </div>
    {:else if !isRegistered}
      <!-- Login Form -->
      <div style="max-width: 500px; margin: 0 auto;">
        <div style="background: white; border-radius: 16px; padding: 2rem; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
          <div style="text-align: center; margin-bottom: 2rem;">
            <div style="font-size: 3rem; margin-bottom: 0.5rem;">🔐</div>
            <h3 style="font-size: 1.5rem; font-weight: 700; color: #111827;">Log Masuk Untuk Mengundi</h3>
          </div>

          {#if error}
            <div style="background: #fef2f2; border-left: 4px solid #ef4444; color: #991b1b; padding: 1rem; border-radius: 8px; margin-bottom: 1.5rem;">
              ⚠️ {error}
            </div>
          {/if}

          <form on:submit|preventDefault={handleLogin}>
            <div style="margin-bottom: 1.5rem;">
              <label style="display: block; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
                Email Anda
              </label>
              <input
                type="email"
                bind:value={voterEmail}
                placeholder="email@example.com"
                required
                style="width: 100%; padding: 0.875rem; border: 2px solid #d1d5db; border-radius: 12px; font-size: 1rem; font-family: inherit;"
              />
            </div>

            <button
              type="submit"
              style="width: 100%; background: linear-gradient(135deg, #14b8a6, #3b82f6); color: white; padding: 1rem; border-radius: 12px; font-size: 1.125rem; font-weight: 700; border: none; cursor: pointer; font-family: inherit;"
            >
              🔐 Log Masuk
            </button>
          </form>

          <div style="margin-top: 1.5rem; padding-top: 1.5rem; border-top: 1px solid #e5e7eb; text-align: center;">
            <p style="color: #6b7280; font-size: 0.875rem;">
              Hanya pengundi berdaftar boleh mengundi
            </p>
            <a href="/" style="display: inline-block; margin-top: 1rem; color: #3b82f6; text-decoration: none; font-weight: 600;">
              ← Kembali ke Halaman Utama
            </a>
          </div>
        </div>
      </div>
    {:else if hasVoted}
      <!-- Already Voted -->
      <div style="background: white; border-radius: 16px; padding: 3rem; text-align: center; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
        <div style="font-size: 4rem; margin-bottom: 1rem;">✅</div>
        <h3 style="font-size: 1.75rem; font-weight: 700; color: #10b981; margin-bottom: 0.5rem;">
          Undi Telah Direkodkan
        </h3>
        <p style="color: #6b7280; margin-bottom: 1.5rem;">
          Terima kasih, {voterName}!
        </p>
        <p style="color: #9ca3af;">
          Anda telah mengundi. Setiap pengundi hanya boleh mengundi sekali sahaja.
        </p>
        <a href="/" style="display: inline-block; margin-top: 2rem; color: #3b82f6; text-decoration: none; font-weight: 600;">
          ← Kembali ke Halaman Utama
        </a>
      </div>
    {:else if success}
      <!-- Success Message -->
      <div style="background: white; border-radius: 16px; padding: 3rem; text-align: center; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
        <div style="font-size: 4rem; margin-bottom: 1rem;">🎉</div>
        <h3 style="font-size: 1.75rem; font-weight: 700; color: #10b981; margin-bottom: 0.5rem;">
          Terima Kasih!
        </h3>
        <p style="color: #6b7280; margin-bottom: 1.5rem;">
          {success}
        </p>
        <a href="/" style="display: inline-block; color: #3b82f6; text-decoration: none; font-weight: 600;">
          ← Kembali ke Halaman Utama
        </a>
      </div>
    {:else}
      <!-- Voting Form -->
      <div>
        <!-- Welcome Message -->
        <div style="background: linear-gradient(135deg, #eff6ff, #dbeafe); border: 2px solid #93c5fd; border-radius: 16px; padding: 1.5rem; margin-bottom: 2rem;">
          <p style="font-size: 1.125rem; color: #1e40af;">
            <strong>Selamat datang, {voterName}!</strong><br>
            <span style="font-size: 0.875rem; color: #3b82f6;">Email: {voterEmail}</span>
          </p>
        </div>

        {#if error}
          <div style="background: #fef2f2; border-left: 4px solid #ef4444; color: #991b1b; padding: 1rem; border-radius: 8px; margin-bottom: 1.5rem;">
            ⚠️ {error}
          </div>
        {/if}

        <!-- Voting Instructions -->
        <div style="background: #fefce8; border-left: 4px solid #fbbf24; padding: 1rem; border-radius: 8px; margin-bottom: 2rem;">
          <p style="color: #92400e; font-weight: 600; margin-bottom: 0.5rem;">📋 Arahan:</p>
          <ul style="color: #78350f; margin-left: 1.5rem; line-height: 1.8;">
            <li>Pilih SATU calon untuk setiap jawatan</li>
            <li>Anda boleh skip jawatan yang tidak mahu dipilih</li>
            <li>Klik "Hantar Undi" apabila selesai</li>
            <li>Undi tidak boleh diubah selepas dihantar</li>
          </ul>
        </div>

        <!-- Positions & Candidates -->
        {#each enabledPositions as position}
          <div style="background: white; border-radius: 16px; padding: 1.5rem; margin-bottom: 1.5rem; box-shadow: 0 2px 4px rgba(0,0,0,0.1); border: 2px solid {votes[position.position] ? '#10b981' : '#e5e7eb'};">
            
            <!-- Position Header -->
            <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 1.5rem; padding-bottom: 1rem; border-bottom: 2px solid #e5e7eb;">
              <div style="display: flex; align-items: center; gap: 0.75rem;">
                <span style="font-size: 2rem;">{getPositionIcon(position.position)}</span>
                <h3 style="font-size: 1.5rem; font-weight: 700; color: #111827;">
                  {position.position}
                </h3>
              </div>
              {#if votes[position.position]}
                <span style="background: #10b981; color: white; padding: 0.5rem 1rem; border-radius: 9999px; font-size: 0.875rem; font-weight: 600;">
                  ✓ Dipilih
                </span>
              {/if}
            </div>

            <!-- Candidates -->
            {#if position.candidates.length === 0}
              <p style="text-align: center; color: #9ca3af; padding: 1rem; font-style: italic;">
                Tiada calon untuk jawatan ini
              </p>
            {:else}
              <div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(250px, 1fr)); gap: 1rem;">
                {#each position.candidates as candidate}
                  <button
                    type="button"
                    on:click={() => selectCandidate(position.position, candidate.candidate_name)}
                    style="
                      padding: 1.25rem;
                      border: 3px solid {votes[position.position] === candidate.candidate_name ? '#10b981' : '#e5e7eb'};
                      background: {votes[position.position] === candidate.candidate_name ? '#d1fae5' : 'white'};
                      border-radius: 12px;
                      cursor: pointer;
                      transition: all 0.2s;
                      text-align: left;
                      font-family: inherit;
                    "
                    on:mouseenter={(e) => {
                      if (votes[position.position] !== candidate.candidate_name) {
                        e.currentTarget.style.borderColor = '#93c5fd';
                        e.currentTarget.style.background = '#eff6ff';
                      }
                    }}
                    on:mouseleave={(e) => {
                      if (votes[position.position] !== candidate.candidate_name) {
                        e.currentTarget.style.borderColor = '#e5e7eb';
                        e.currentTarget.style.background = 'white';
                      }
                    }}
                  >
                    <div style="display: flex; align-items: center; gap: 0.75rem; margin-bottom: 0.5rem;">
                      <span style="font-size: 1.5rem;">
                        {votes[position.position] === candidate.candidate_name ? '✅' : '👤'}
                      </span>
                      <span style="font-weight: 700; font-size: 1.125rem; color: #111827; text-transform: uppercase;">
                        {candidate.candidate_name}
                      </span>
                    </div>
                  </button>
                {/each}
              </div>
            {/if}
          </div>
        {/each}

        <!-- Submit Button -->
        <div style="position: sticky; bottom: 0; background: white; padding: 1.5rem; border-radius: 16px; box-shadow: 0 -4px 6px rgba(0,0,0,0.1); margin-top: 2rem;">
          <button
            on:click={handleSubmitVotes}
            disabled={voting}
            style="
              width: 100%;
              background: linear-gradient(135deg, #10b981, #14b8a6);
              color: white;
              padding: 1.25rem;
              border-radius: 16px;
              font-size: 1.25rem;
              font-weight: 700;
              border: none;
              cursor: pointer;
              font-family: inherit;
              {voting ? 'opacity: 0.6; cursor: not-allowed;' : ''}
            "
          >
            {voting ? '⏳ Menghantar...' : '🗳️ Hantar Undi'}
          </button>
        </div>
      </div>
    {/if}

  </div>
</div>