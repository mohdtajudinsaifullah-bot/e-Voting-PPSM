<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let loading = true;
  let results = [];
  let totalVoters = 0;
  let error = '';

  onMount(() => {
    loadResults();
  });

  async function loadResults() {
    loading = true;
    try {
      // Get all votes
      const { data: votes, error: votesError } = await supabase
        .from('votes')
        .select('*');

      if (votesError) throw votesError;

      // Count unique voters
      const uniqueVoters = new Set(votes.map(v => v.voter_email));
      totalVoters = uniqueVoters.size;

      // Get enabled positions to maintain order
      const { data: positions } = await supabase
        .from('position_settings')
        .select('*')
        .eq('is_enabled', true)
        .order('position');

      // Process votes by position
      const positionResults = {};
      
      votes.forEach(vote => {
        if (!positionResults[vote.position]) {
          positionResults[vote.position] = {};
        }
        
        if (!positionResults[vote.position][vote.candidate_name]) {
          positionResults[vote.position][vote.candidate_name] = 0;
        }
        
        positionResults[vote.position][vote.candidate_name]++;
      });

      // Format results
      results = positions.map(pos => {
        const candidateVotes = positionResults[pos.position] || {};
        
        // Convert to array and sort by votes
        const candidates = Object.entries(candidateVotes)
          .map(([name, votes]) => ({ name, votes }))
          .sort((a, b) => b.votes - a.votes);

        const totalVotes = candidates.reduce((sum, c) => sum + c.votes, 0);

        return {
          position: pos.position,
          candidates,
          totalVotes
        };
      });

    } catch (err) {
      console.error('Error loading results:', err);
      error = 'Ralat memuatkan keputusan: ' + err.message;
    } finally {
      loading = false;
    }
  }

  function getPositionIcon(position) {
    if (position.includes('Presiden')) return '👑';
    if (position.includes('Exco')) return '💼';
    if (position.includes('Pemeriksa')) return '📊';
    return '🎯';
  }

  function getPercentage(votes, total) {
    if (total === 0) return 0;
    return ((votes / total) * 100).toFixed(1);
  }
</script>

<svelte:head>
  <title>Keputusan Pengundian - E-VOTING PPSM</title>
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

<!-- NO SIDEBAR - Full Width Layout with WHITE Background -->
<div style="min-height: 100vh; background: #ffffff;">
  
  <!-- Header -->
  <div style="background: linear-gradient(135deg, #14b8a6 0%, #06b6d4 50%, #3b82f6 100%); padding: 2rem 1rem; color: white; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
    <div style="max-width: 1200px; margin: 0 auto;">
      <div style="text-align: center;">
        <h1 style="font-size: 3rem; font-weight: 700; text-shadow: 2px 2px 4px rgba(0,0,0,0.2); letter-spacing: 0.05em; margin-bottom: 0.5rem;">
          e-VOTING PPSM
        </h1>
        <p style="font-size: 1.125rem; opacity: 0.95;">Keputusan Pengundian</p>
      </div>
    </div>
  </div>

  <!-- Main Content - Full Width, White Background -->
  <div style="max-width: 1200px; margin: 0 auto; padding: 2rem;">
    
    {#if loading}
      <div style="text-align: center; padding: 3rem; background: white; border-radius: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
        <div style="font-size: 3rem; margin-bottom: 1rem;">⏳</div>
        <p style="color: #6b7280; font-size: 1.125rem;">Memuatkan keputusan...</p>
      </div>
    {:else if error}
      <div style="background: #fef2f2; border-left: 4px solid #ef4444; color: #991b1b; padding: 1.5rem; border-radius: 8px;">
        ⚠️ {error}
      </div>
    {:else}
      <!-- Summary Stats -->
      <div style="background: white; border-radius: 16px; padding: 2rem; margin-bottom: 2rem; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
        <h2 style="font-size: 1.75rem; font-weight: 700; color: #111827; margin-bottom: 1.5rem;">
          📊 Keputusan Pengundian
        </h2>
        <p style="color: #6b7280; font-size: 1rem; margin-bottom: 1rem;">
          Keputusan dikira berdasarkan undi yang telah diterima
        </p>
        <div style="background: linear-gradient(135deg, #eff6ff, #dbeafe); border: 2px solid #93c5fd; border-radius: 12px; padding: 1.25rem; display: inline-block;">
          <p style="font-size: 0.875rem; color: #1e40af; margin-bottom: 0.25rem;">Jumlah Pengundi</p>
          <p style="font-size: 2rem; font-weight: 700; color: #1e3a8a;">{totalVoters} orang</p>
        </div>
      </div>

      <!-- Results by Position -->
      {#each results as result}
        <div style="background: white; border-radius: 16px; padding: 2rem; margin-bottom: 1.5rem; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
          
          <!-- Position Header -->
          <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 1.5rem; padding-bottom: 1rem; border-bottom: 2px solid #e5e7eb;">
            <div style="display: flex; align-items: center; gap: 0.75rem;">
              <span style="font-size: 2rem;">{getPositionIcon(result.position)}</span>
              <h3 style="font-size: 1.5rem; font-weight: 700; color: #111827;">
                {result.position}
              </h3>
            </div>
            <span style="background: #0ea5e9; color: white; padding: 0.5rem 1rem; border-radius: 9999px; font-size: 0.875rem; font-weight: 600;">
              {result.totalVotes} undi
            </span>
          </div>

          <!-- Candidates Results -->
          {#if result.candidates.length === 0}
            <p style="text-align: center; color: #9ca3af; padding: 2rem; font-style: italic;">
              Tiada undi untuk jawatan ini
            </p>
          {:else}
            <div style="display: flex; flex-direction: column; gap: 1rem;">
              {#each result.candidates as candidate, index}
                <div style="background: {index === 0 ? '#d1fae5' : '#f9fafb'}; border: 2px solid {index === 0 ? '#10b981' : '#e5e7eb'}; border-radius: 12px; padding: 1.5rem; position: relative; overflow: hidden;">
                  
                  <!-- Winner Badge -->
                  {#if index === 0 && result.candidates.length > 1}
                    <div style="position: absolute; top: 0; right: 0; background: linear-gradient(135deg, #10b981, #14b8a6); color: white; padding: 0.5rem 1.5rem; border-bottom-left-radius: 12px; font-size: 0.75rem; font-weight: 700; letter-spacing: 0.05em;">
                      🏆 PEMENANG
                    </div>
                  {/if}

                  <div style="display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 1rem;">
                    
                    <!-- Candidate Info -->
                    <div style="flex: 1; min-width: 200px;">
                      <div style="display: flex; align-items: center; gap: 0.75rem; margin-bottom: 0.5rem;">
                        <span style="font-size: 1.75rem;">
                          {index === 0 ? '🥇' : index === 1 ? '🥈' : index === 2 ? '🥉' : '👤'}
                        </span>
                        <h4 style="font-size: 1.25rem; font-weight: 700; color: #111827; text-transform: uppercase;">
                          {candidate.name}
                        </h4>
                      </div>
                      
                      <!-- Vote Bar -->
                      <div style="background: #e5e7eb; height: 8px; border-radius: 9999px; overflow: hidden; margin-top: 0.75rem;">
                        <div style="background: linear-gradient(90deg, #10b981, #14b8a6); height: 100%; width: {getPercentage(candidate.votes, result.totalVotes)}%; transition: width 0.5s;"></div>
                      </div>
                    </div>

                    <!-- Vote Count -->
                    <div style="text-align: right;">
                      <div style="background: {index === 0 ? '#10b981' : '#6b7280'}; color: white; padding: 0.75rem 1.5rem; border-radius: 12px; display: inline-flex; flex-direction: column; align-items: center; gap: 0.25rem; min-width: 100px;">
                        <span style="font-size: 1.75rem; font-weight: 700;">{candidate.votes}</span>
                        <span style="font-size: 0.75rem; opacity: 0.9;">undi</span>
                      </div>
                      <p style="color: #6b7280; font-size: 0.875rem; margin-top: 0.5rem; font-weight: 600;">
                        {getPercentage(candidate.votes, result.totalVotes)}%
                      </p>
                    </div>
                  </div>
                </div>
              {/each}
            </div>
          {/if}
        </div>
      {/each}

      {#if results.length === 0}
        <div style="background: white; border-radius: 16px; padding: 3rem; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
          <div style="font-size: 3rem; margin-bottom: 1rem;">📭</div>
          <p style="color: #6b7280; font-size: 1.125rem;">Tiada keputusan untuk dipaparkan</p>
        </div>
      {/if}

      <!-- Back Button -->
      <div style="text-align: center; margin-top: 2rem;">
        <a href="/" style="display: inline-block; background: #3b82f6; color: white; padding: 1rem 2rem; border-radius: 12px; text-decoration: none; font-weight: 600; transition: all 0.2s;"
           on:mouseenter={(e) => e.currentTarget.style.background = '#2563eb'}
           on:mouseleave={(e) => e.currentTarget.style.background = '#3b82f6'}>
          ← Kembali ke Halaman Utama
        </a>
      </div>
    {/if}

  </div>
</div>