<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';

  let settings = null;
  let loading = true;
  let error = '';
  let success = '';
  let saving = false;

  let startDate = '';
  let startTime = '';
  let endDate = '';
  let endTime = '';
  let isActive = false;

  onMount(() => {
    loadSettings();
  });

  async function loadSettings() {
    loading = true;
    try {
      const { data, error: dbError } = await supabase
        .from('voting_settings')
        .select('*')
        .order('created_at', { ascending: false })
        .limit(1)
        .single();

      if (data) {
        settings = data;
        
        const start = new Date(data.start_date);
        const end = new Date(data.end_date);
        
        startDate = start.toISOString().split('T')[0];
        startTime = start.toTimeString().slice(0, 5);
        endDate = end.toISOString().split('T')[0];
        endTime = end.toTimeString().slice(0, 5);
        isActive = data.is_active;
      }
    } catch (err) {
      console.log('No settings found');
    } finally {
      loading = false;
    }
  }

  async function handleSave() {
    if (!startDate || !startTime || !endDate || !endTime) {
      error = 'Sila lengkapkan semua maklumat tarikh dan masa';
      return;
    }

    const startDateTime = new Date(`${startDate}T${startTime}:00`);
    const endDateTime = new Date(`${endDate}T${endTime}:00`);

    if (endDateTime <= startDateTime) {
      error = 'Tarikh tamat mesti selepas tarikh mula';
      return;
    }

    saving = true;
    error = '';

    try {
      const data = {
        start_date: startDateTime.toISOString(),
        end_date: endDateTime.toISOString(),
        is_active: isActive,
        updated_at: new Date().toISOString()
      };

      if (settings) {
        const { error: dbError } = await supabase
          .from('voting_settings')
          .update(data)
          .eq('id', settings.id);

        if (dbError) throw dbError;
      } else {
        const { data: inserted, error: dbError } = await supabase
          .from('voting_settings')
          .insert([data])
          .select()
          .single();

        if (dbError) throw dbError;
        settings = inserted;
      }

      success = 'Tetapan pengundian berjaya disimpan!';
      setTimeout(() => success = '', 3000);
      
    } catch (err) {
      error = 'Gagal simpan tetapan: ' + err.message;
    } finally {
      saving = false;
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

  function getVotingStatus() {
    if (!settings) return { text: 'Belum Ditetapkan', color: '#6b7280', icon: '⚪' };
    
    const now = new Date();
    const start = new Date(settings.start_date);
    const end = new Date(settings.end_date);

    if (!settings.is_active) {
      return { text: 'Tidak Aktif', color: '#ef4444', icon: '🔴' };
    }

    if (now < start) {
      return { text: 'Belum Bermula', color: '#f59e0b', icon: '🟡' };
    }

    if (now >= start && now <= end) {
      return { text: 'AKTIF - Sedang Berlangsung', color: '#10b981', icon: '🟢' };
    }

    if (now > end) {
      return { text: 'Tamat', color: '#6b7280', icon: '⚫' };
    }

    return { text: 'Unknown', color: '#6b7280', icon: '⚪' };
  }

  $: status = getVotingStatus();
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
    <h1 style="font-size: 2rem; font-weight: 700; color: #111827; margin-bottom: 0.5rem;">
      📅 Masa Pengundian
    </h1>
    <p style="color: #6b7280;">Tetapkan tarikh dan masa tempoh pengundian</p>
  </div>

  {#if success}
    <div style="background: #d1fae5; border-left: 4px solid #10b981; color: #065f46; padding: 1rem; border-radius: 8px; margin-bottom: 1rem;">
      ✅ {success}
    </div>
  {/if}

  {#if error}
    <div style="background: #fee2e2; border-left: 4px solid #ef4444; color: #991b1b; padding: 1rem; border-radius: 8px; margin-bottom: 1rem;">
      ⚠️ {error}
    </div>
  {/if}

  <!-- Current Status -->
  <div style="background: white; border-radius: 16px; padding: 2rem; margin-bottom: 2rem; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
    <div style="text-align: center;">
      <div style="font-size: 4rem; margin-bottom: 0.5rem;">{status.icon}</div>
      <div style="font-size: 1.75rem; font-weight: 700; color: {status.color}; margin-bottom: 1rem;">
        {status.text}
      </div>
      
      {#if settings}
        <div style="color: #6b7280; line-height: 1.8;">
          <div><strong>Mula:</strong> {formatDateTime(settings.start_date)}</div>
          <div><strong>Tamat:</strong> {formatDateTime(settings.end_date)}</div>
        </div>
      {:else}
        <p style="color: #9ca3af; font-style: italic;">
          Tiada tempoh pengundian ditetapkan
        </p>
      {/if}
    </div>
  </div>

  <!-- Settings Form -->
  {#if loading}
    <div style="text-align: center; padding: 3rem; color: #6b7280;">
      ⏳ Memuatkan tetapan...
    </div>
  {:else}
    <div style="background: white; border-radius: 16px; padding: 2rem; box-shadow: 0 1px 3px rgba(0,0,0,0.1); margin-bottom: 2rem;">
      <h2 style="font-size: 1.5rem; font-weight: 700; color: #111827; margin-bottom: 1.5rem;">
        ⚙️ Tetapan Pengundian
      </h2>

      <form on:submit|preventDefault={handleSave}>
        
        <!-- Start Date & Time -->
        <div style="margin-bottom: 2rem; padding: 1.5rem; background: #f0fdf4; border-radius: 12px; border: 2px solid #86efac;">
          <h3 style="font-size: 1.25rem; font-weight: 600; color: #065f46; margin-bottom: 1rem; display: flex; align-items: center; gap: 0.5rem;">
            🟢 Tarikh & Masa Mula Pengundian
          </h3>
          
          <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 1rem;">
            <div>
              <label style="display: block; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
                Tarikh Mula
              </label>
              <input
                type="date"
                bind:value={startDate}
                required
                style="width: 100%; padding: 0.875rem; border: 2px solid #d1d5db; border-radius: 8px; font-size: 1rem; font-family: inherit;"
              />
            </div>
            <div>
              <label style="display: block; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
                Masa Mula
              </label>
              <input
                type="time"
                bind:value={startTime}
                required
                style="width: 100%; padding: 0.875rem; border: 2px solid #d1d5db; border-radius: 8px; font-size: 1rem; font-family: inherit;"
              />
            </div>
          </div>
        </div>

        <!-- End Date & Time -->
        <div style="margin-bottom: 2rem; padding: 1.5rem; background: #fef2f2; border-radius: 12px; border: 2px solid #fca5a5;">
          <h3 style="font-size: 1.25rem; font-weight: 600; color: #991b1b; margin-bottom: 1rem; display: flex; align-items: center; gap: 0.5rem;">
            🔴 Tarikh & Masa Tamat Pengundian
          </h3>
          
          <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 1rem;">
            <div>
              <label style="display: block; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
                Tarikh Tamat
              </label>
              <input
                type="date"
                bind:value={endDate}
                required
                style="width: 100%; padding: 0.875rem; border: 2px solid #d1d5db; border-radius: 8px; font-size: 1rem; font-family: inherit;"
              />
            </div>
            <div>
              <label style="display: block; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
                Masa Tamat
              </label>
              <input
                type="time"
                bind:value={endTime}
                required
                style="width: 100%; padding: 0.875rem; border: 2px solid #d1d5db; border-radius: 8px; font-size: 1rem; font-family: inherit;"
              />
            </div>
          </div>
        </div>

        <!-- Active Toggle -->
        <div style="margin-bottom: 2rem; padding: 1.5rem; background: #eff6ff; border-radius: 12px; border: 2px solid #93c5fd;">
          <label style="display: flex; align-items: center; gap: 1rem; cursor: pointer;">
            <input
              type="checkbox"
              bind:checked={isActive}
              style="width: 1.5rem; height: 1.5rem; cursor: pointer;"
            />
            <div>
              <div style="font-weight: 600; color: #1e40af; font-size: 1.125rem;">
                Aktifkan Pengundian
              </div>
              <div style="color: #6b7280; font-size: 0.875rem; margin-top: 0.25rem;">
                Tandakan untuk membenarkan pengundian dalam tempoh yang ditetapkan
              </div>
            </div>
          </label>
        </div>

        <!-- Save Button -->
        <button
          type="submit"
          disabled={saving}
          style="width: 100%; background: linear-gradient(135deg, #1e3a8a, #3b82f6); color: white; padding: 1.25rem; border-radius: 12px; font-size: 1.25rem; font-weight: 700; border: none; cursor: pointer; transition: all 0.2s; font-family: inherit; {saving ? 'opacity: 0.6; cursor: not-allowed;' : ''}"
        >
          {saving ? '⏳ Menyimpan...' : '💾 Simpan Tetapan'}
        </button>

      </form>

      <!-- Info Box -->
      <div style="margin-top: 2rem; padding: 1rem; background: #fefce8; border-left: 4px solid #fbbf24; border-radius: 8px;">
        <div style="font-weight: 600; color: #92400e; margin-bottom: 0.5rem;">
          ℹ️ Nota Penting:
        </div>
        <ul style="color: #78350f; margin-left: 1.5rem; line-height: 1.8;">
          <li>Pengundi hanya boleh mengundi dalam tempoh yang ditetapkan</li>
          <li>Pastikan tarikh dan masa adalah betul sebelum mengaktifkan</li>
          <li>Anda boleh ubah tetapan pada bila-bila masa</li>
          <li>Jika pengundian tidak aktif, tiada sesiapa boleh undi walaupun dalam tempoh</li>
        </ul>
      </div>
    </div>
  {/if}
</div>