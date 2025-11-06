<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let nominations = [];
  let loading = true;
  let error = '';

  const positionGroups = [
    {
      title: 'Presiden',
      positions: ['president', 'Presiden'],
      icon: '👑'
    },
    {
      title: 'Timbalan Presiden',
      positions: ['vice_president', 'Timbalan Presiden'],
      icon: '🎖️'
    },
    {
      title: 'Naib Presiden',
      positions: [
        'deputy_president_1', 'deputy_president_2', 
        'Naib Presiden 1', 'Naib Presiden 2',
        'Naib Presiden I', 'Naib Presiden II'
      ],
      icon: '⭐'
    },
    {
      title: 'Exco',
      positions: [
        'exco_1', 'exco_2', 'exco_3', 'exco_4', 'exco_5', 'exco_6', 'exco_7',
        'Exco 1', 'Exco 2', 'Exco 3', 'Exco 4', 'Exco 5', 'Exco 6', 'Exco 7'
      ],
      icon: '👥'
    },
    {
      title: 'Pemeriksa Kira-kira',
      positions: [
        'auditor_1', 'auditor_2',
        'Pemeriksa Kira-kira 1', 'Pemeriksa Kira-kira 2'
      ],
      icon: '📊'
    }
  ];

  let groupedNominations = {};

  onMount(async () => {
    await fetchNominations();
  });

  async function fetchNominations() {
    loading = true;
    error = '';

    try {
      const { data, error: dbError } = await supabase
        .from('nominations')
        .select('*')
        .order('nominated_at', { ascending: false });

      if (dbError) throw dbError;

      nominations = data || [];
      
      // Group nominations
      groupedNominations = {};
      
      positionGroups.forEach(group => {
        groupedNominations[group.title] = nominations.filter(nom => 
          group.positions.includes(nom.position)
        );
      });

    } catch (err) {
      error = 'Gagal memuatkan senarai pencalonan: ' + err.message;
      console.error('Error:', err);
    } finally {
      loading = false;
    }
  }

  function getNomineeCount(groupTitle) {
    return groupedNominations[groupTitle]?.length || 0;
  }

  // Get nominees with nomination count - FIXED to use candidate_name
  function getNomineesWithCount(groupTitle) {
    const noms = groupedNominations[groupTitle] || [];
    
    // Count nominations per candidate
    const countMap = new Map();
    noms.forEach(nom => {
      // Use candidate_name field (correct field name from database)
      const name = nom.candidate_name || 'Tiada Nama';
      
      if (countMap.has(name)) {
        countMap.set(name, {
          ...countMap.get(name),
          count: countMap.get(name).count + 1
        });
      } else {
        countMap.set(name, {
          ...nom,
          candidate_name: name,
          count: 1
        });
      }
    });
    
    // Convert to array and sort by count (descending)
    return Array.from(countMap.values()).sort((a, b) => b.count - a.count);
  }
</script>

<svelte:head>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
  </style>
</svelte:head>

<div style="min-height: 100vh; background: linear-gradient(135deg, #14b8a6 0%, #06b6d4 50%, #3b82f6 100%); padding: 2rem 1rem;">
  <div style="max-width: 1000px; margin: 0 auto;">
    
    <!-- Header -->
    <div style="text-align: center; margin-bottom: 2rem;">
      <h1 style="font-size: 3rem; font-weight: 700; color: white; text-shadow: 2px 2px 4px rgba(0,0,0,0.2); letter-spacing: 0.05em; margin-bottom: 0.5rem;">
        e-VOTING PPSM
      </h1>
      <p style="font-size: 1.25rem; color: white; text-shadow: 1px 1px 2px rgba(0,0,0,0.2);">
        Senarai Pencalonan
      </p>
    </div>

    {#if loading}
      <!-- Loading -->
      <div style="background: white; border-radius: 24px; box-shadow: 0 20px 60px rgba(0,0,0,0.2); padding: 4rem; text-align: center;">
        <div style="width: 3rem; height: 3rem; border: 4px solid #3b82f6; border-top-color: transparent; border-radius: 50%; animation: spin 1s linear infinite; margin: 0 auto 1rem;"></div>
        <p style="color: #6b7280; font-size: 1.125rem;">Memuatkan data...</p>
      </div>
    {:else if error}
      <!-- Error -->
      <div style="background: white; border-radius: 24px; box-shadow: 0 20px 60px rgba(0,0,0,0.2); padding: 3rem; text-align: center;">
        <div style="font-size: 4rem; margin-bottom: 1rem;">⚠️</div>
        <h2 style="font-size: 1.5rem; font-weight: 700; color: #ef4444; margin-bottom: 1rem;">Ralat</h2>
        <p style="color: #6b7280;">{error}</p>
      </div>
    {:else}
      <!-- Nominations List -->
      <div style="background: white; border-radius: 24px; box-shadow: 0 20px 60px rgba(0,0,0,0.2); padding: 2.5rem;">
        
        <div style="margin-bottom: 2rem;">
          <h2 style="font-size: 2rem; font-weight: 700; color: #111827; margin-bottom: 0.5rem;">Senarai Pencalonan</h2>
          <p style="color: #6b7280;">Senarai calon yang telah dicalonkan untuk setiap jawatan</p>
        </div>

        <div style="display: flex; flex-direction: column; gap: 1.5rem;">
          
          {#each positionGroups as group}
            {@const nominees = getNomineesWithCount(group.title)}
            {@const count = getNomineeCount(group.title)}
            
            <div style="background: linear-gradient(to right, #f0fdfa, #ecfeff); border: 2px solid #a5f3fc; border-radius: 16px; padding: 1.5rem; transition: all 0.2s;">
              
              <!-- Position Header -->
              <div style="display: flex; align-items: center; justify-content: space-between; margin-bottom: 1rem;">
                <div style="display: flex; align-items: center; gap: 0.75rem;">
                  <span style="font-size: 1.75rem;">{group.icon}</span>
                  <h3 style="font-size: 1.5rem; font-weight: 700; color: #0f172a;">{group.title}</h3>
                </div>
                <span style="background: linear-gradient(135deg, #06b6d4, #0891b2); color: white; padding: 0.5rem 1rem; border-radius: 9999px; font-size: 0.875rem; font-weight: 600; box-shadow: 0 2px 8px rgba(8, 145, 178, 0.3);">
                  {count} pencalonan
                </span>
              </div>

              <!-- Nominees List -->
              {#if nominees.length > 0}
                <div style="display: flex; flex-direction: column; gap: 0.75rem;">
                  {#each nominees as nominee}
                    <div style="background: white; border: 2px solid #e0f2fe; border-radius: 12px; padding: 1.25rem 1.5rem; display: flex; align-items: center; justify-content: space-between; gap: 1rem; box-shadow: 0 2px 4px rgba(0,0,0,0.05); transition: all 0.2s;"
                         on:mouseenter={(e) => {
                           e.currentTarget.style.borderColor = '#06b6d4';
                           e.currentTarget.style.boxShadow = '0 4px 12px rgba(6, 182, 212, 0.15)';
                         }}
                         on:mouseleave={(e) => {
                           e.currentTarget.style.borderColor = '#e0f2fe';
                           e.currentTarget.style.boxShadow = '0 2px 4px rgba(0,0,0,0.05)';
                         }}>
                      
                      <!-- Candidate Name and Details (Left) -->
                      <div style="flex: 1; min-width: 0;">
                        <!-- Name (using candidate_name) -->
                        <h4 style="font-weight: 700; color: #0f172a; font-size: 1.5rem; margin-bottom: 0.375rem; text-transform: uppercase; letter-spacing: 0.025em; line-height: 1.2; word-break: break-word;">
                          {nominee.candidate_name}
                        </h4>
                        <!-- Details -->
                        {#if nominee.candidate_email || nominee.candidate_state || nominee.candidate_department}
                          <div style="color: #64748b; font-size: 0.875rem; display: flex; flex-wrap: gap: 0.75rem; line-height: 1.5;">
                            {#if nominee.candidate_email}
                              <span style="display: flex; align-items: center; gap: 0.25rem;">
                                📧 {nominee.candidate_email}
                              </span>
                            {/if}
                            {#if nominee.candidate_state}
                              <span style="display: flex; align-items: center; gap: 0.25rem;">
                                📍 {nominee.candidate_state}
                              </span>
                            {/if}
                            {#if nominee.candidate_department}
                              <span style="display: flex; align-items: center; gap: 0.25rem;">
                                🏢 {nominee.candidate_department}
                              </span>
                            {/if}
                          </div>
                        {/if}
                      </div>

                      <!-- Nomination Count Badge (Right - Green) -->
                      <div style="background: linear-gradient(135deg, #10b981, #059669); color: white; padding: 0.625rem 1.25rem; border-radius: 9999px; font-size: 1rem; font-weight: 700; flex-shrink: 0; box-shadow: 0 2px 8px rgba(16, 185, 129, 0.35); display: flex; align-items: center; gap: 0.5rem; white-space: nowrap;">
                        <span style="font-size: 1.25rem;">🗳️</span>
                        <span>{nominee.count} pencalonan</span>
                      </div>
                    </div>
                  {/each}
                </div>
              {:else}
                <div style="background: #fef2f2; border: 2px dashed #fecaca; border-radius: 12px; padding: 2rem; text-align: center;">
                  <p style="color: #991b1b; font-weight: 600;">Tiada pencalonan lagi untuk jawatan ini</p>
                </div>
              {/if}
            </div>
          {/each}

        </div>

        <!-- Total Summary -->
        <div style="margin-top: 2rem; padding-top: 2rem; border-top: 2px solid #e5e7eb;">
          <div style="background: linear-gradient(135deg, #dbeafe, #bfdbfe); border: 2px solid #60a5fa; border-radius: 12px; padding: 1.5rem; text-align: center;">
            <div style="font-size: 2.5rem; font-weight: 700; color: #1e40af; margin-bottom: 0.5rem;">
              {nominations.length}
            </div>
            <div style="color: #1e3a8a; font-weight: 600; font-size: 1.125rem;">
              Jumlah Pencalonan Keseluruhan
            </div>
          </div>
        </div>
      </div>

      <!-- Action Buttons -->
      <div style="display: flex; gap: 1rem; margin-top: 2rem;">
        <a href="/nomination" style="flex: 1; background: linear-gradient(135deg, #14b8a6, #3b82f6); color: white; padding: 1rem 2rem; border-radius: 16px; text-decoration: none; text-align: center; font-weight: 700; font-size: 1.125rem; box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3); transition: all 0.2s;"
           on:mouseenter={(e) => {
             e.target.style.transform = 'translateY(-2px)';
             e.target.style.boxShadow = '0 6px 20px rgba(59, 130, 246, 0.4)';
           }}
           on:mouseleave={(e) => {
             e.target.style.transform = 'translateY(0)';
             e.target.style.boxShadow = '0 4px 12px rgba(59, 130, 246, 0.3)';
           }}>
          ➕ Tambah Pencalonan
        </a>
        <button on:click={fetchNominations} style="padding: 1rem 2rem; background: white; border: 2px solid #d1d5db; color: #374151; border-radius: 16px; font-weight: 700; font-size: 1.125rem; cursor: pointer; transition: all 0.2s;"
                on:mouseenter={(e) => {
                  e.target.style.background = '#f9fafb';
                  e.target.style.borderColor = '#9ca3af';
                }}
                on:mouseleave={(e) => {
                  e.target.style.background = 'white';
                  e.target.style.borderColor = '#d1d5db';
                }}>
          🔄 Refresh
        </button>
      </div>
    {/if}

    <!-- Back Link -->
    <div style="text-align: center; margin-top: 2rem;">
      <a href="/" style="color: white; font-weight: 600; font-size: 1.125rem; text-decoration: none; text-shadow: 1px 1px 2px rgba(0,0,0,0.3); transition: color 0.2s;">
        ← Kembali ke Halaman Utama
      </a>
    </div>
  </div>
</div>

<style>
  @keyframes spin {
    from { transform: rotate(0deg); }
    to { transform: rotate(360deg); }
  }
</style>