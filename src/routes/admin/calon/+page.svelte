<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let nominations = [];
  let loading = true;
  let groupedNominations = {};
  let stats = { totalNominations: 0, positionsFilled: 0, uniqueCandidates: 0 };

  const positions = [
    'Presiden',
    'Timbalan Presiden',
    'Naib Presiden I',
    'Naib Presiden II',
    'Exco 1', 'Exco 2', 'Exco 3', 'Exco 4', 'Exco 5', 'Exco 6', 'Exco 7',
    'Pemeriksa Kira-kira 1',
    'Pemeriksa Kira-kira 2'
  ];

  onMount(() => {
    loadNominations();
  });

  async function loadNominations() {
    loading = true;
    try {
      const { data, error } = await supabase
        .from('nominations')
        .select('*')
        .order('position')
        .order('candidate_name');

      if (error) throw error;
      
      nominations = data || [];
      groupNominations();
      calculateStats();
      
    } catch (err) {
      console.error('Error loading nominations:', err);
    } finally {
      loading = false;
    }
  }

  function groupNominations() {
    groupedNominations = {};
    
    positions.forEach(pos => {
      const noms = nominations.filter(n => n.position === pos);
      const counts = {};
      
      noms.forEach(nom => {
        counts[nom.candidate_name] = (counts[nom.candidate_name] || 0) + 1;
      });
      
      const candidates = Object.keys(counts).map(name => ({
        name,
        count: counts[name],
        nominations: noms.filter(n => n.candidate_name === name)
      })).sort((a, b) => b.count - a.count);
      
      groupedNominations[pos] = candidates;
    });
  }

  function calculateStats() {
    stats.totalNominations = nominations.length;
    stats.positionsFilled = Object.keys(groupedNominations).filter(p => groupedNominations[p].length > 0).length;
    stats.uniqueCandidates = Object.values(groupedNominations).reduce((sum, candidates) => sum + candidates.length, 0);
  }

  function getPositionIcon(position) {
    if (position.includes('Presiden')) return '👑';
    if (position.includes('Exco')) return '💼';
    if (position.includes('Pemeriksa')) return '📊';
    return '🎯';
  }
</script>

<svelte:head>
  <style>
    body {
      overflow-y: auto !important;
    }
  </style>
</svelte:head>

<div style="padding-bottom: 3rem;">
  
  <div style="margin-bottom: 2rem;">
    <h2 style="font-size: 2rem; font-weight: 700; color: #111827; display: flex; align-items: center; gap: 0.75rem; margin-bottom: 0.5rem;">
      🎯 Senarai Calon
    </h2>
    <p style="color: #6b7280;">Calon yang telah dicalonkan untuk setiap jawatan</p>
  </div>

  {#if loading}
    <div style="text-align: center; padding: 3rem; color: #6b7280;">
      ⏳ Memuatkan data...
    </div>
  {:else}
    <!-- Stats -->
    <div style="background: linear-gradient(135deg, #3b82f6, #06b6d4); color: white; padding: 2rem; border-radius: 16px; margin-bottom: 2rem; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
      <div style="display: grid; grid-template-columns: repeat(3, 1fr); gap: 2rem; text-align: center;">
        <div>
          <div style="font-size: 2.5rem; font-weight: 700; margin-bottom: 0.5rem;">{stats.totalNominations}</div>
          <div style="font-size: 1rem; opacity: 0.95;">Jumlah Pencalonan</div>
        </div>
        <div>
          <div style="font-size: 2.5rem; font-weight: 700; margin-bottom: 0.5rem;">{stats.positionsFilled}</div>
          <div style="font-size: 1rem; opacity: 0.95;">Jawatan Dicalonkan</div>
        </div>
        <div>
          <div style="font-size: 2.5rem; font-weight: 700; margin-bottom: 0.5rem;">{stats.uniqueCandidates}</div>
          <div style="font-size: 1rem; opacity: 0.95;">Calon Unik</div>
        </div>
      </div>
    </div>

    <!-- Positions -->
    {#each positions as position}
      {@const candidates = groupedNominations[position] || []}
      
      <div style="background: white; border-radius: 16px; padding: 1.5rem; margin-bottom: 1.5rem; box-shadow: 0 2px 4px rgba(0,0,0,0.1);">
        
        <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 1.5rem; padding-bottom: 1rem; border-bottom: 2px solid #e5e7eb;">
          <div style="display: flex; align-items: center; gap: 0.75rem;">
            <span style="font-size: 1.75rem;">{getPositionIcon(position)}</span>
            <h3 style="font-size: 1.5rem; font-weight: 700; color: #111827;">
              {position}
            </h3>
          </div>
          <span style="background: #3b82f6; color: white; padding: 0.5rem 1rem; border-radius: 9999px; font-size: 0.875rem; font-weight: 600;">
            {candidates.length} calon
          </span>
        </div>

        {#if candidates.length === 0}
          <p style="color: #9ca3af; font-style: italic; padding: 1rem 0; text-align: center;">
            Tiada pencalonan lagi
          </p>
        {:else}
          <div style="display: flex; flex-direction: column; gap: 1rem;">
            {#each candidates as candidate}
              <div style="border: 2px solid #e5e7eb; border-radius: 12px; padding: 1.25rem; background: #f9fafb;">
                <div style="display: flex; justify-content: space-between; align-items: center;">
                  <div>
                    <div style="font-size: 1.25rem; font-weight: 700; color: #111827; text-transform: uppercase; margin-bottom: 0.5rem;">
                      {candidate.name}
                    </div>
                    <div style="color: #6b7280; font-size: 0.875rem;">
                      📥 {candidate.count} pencalonan
                    </div>
                  </div>
                  <button style="background: #10b981; color: white; padding: 0.75rem 1.5rem; border-radius: 12px; font-weight: 600; border: none; cursor: pointer; font-family: inherit;">
                    ✅ Dicalonkan
                  </button>
                </div>

                <details style="margin-top: 1rem;">
                  <summary style="cursor: pointer; color: #3b82f6; font-weight: 600; padding: 0.5rem 0; user-select: none;">
                    👥 Lihat pencadang ({candidate.count})
                  </summary>
                  <div style="margin-top: 0.75rem; padding: 1rem; background: white; border-radius: 8px; border: 1px solid #e5e7eb;">
                    {#each candidate.nominations as nom}
                      <div style="padding: 0.75rem 0; border-bottom: 1px solid #f3f4f6;">
                        <strong style="color: #111827;">{nom.nominator_name}</strong>
                        <span style="color: #6b7280; margin-left: 0.5rem;">• {nom.nominator_position}</span>
                      </div>
                    {/each}
                  </div>
                </details>
              </div>
            {/each}
          </div>
        {/if}
      </div>
    {/each}
  {/if}
</div>