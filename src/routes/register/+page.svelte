<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let formData = {
    fullName: '',
    email: '',
    state: '',
    department: ''
  };

  let loading = false;
  let error = '';
  let success = false;

  const states = [
    'Johor', 'Kedah', 'Kelantan', 'Melaka', 'Negeri Sembilan',
    'Pahang', 'Penang', 'Perak', 'Perlis', 'Sabah',
    'Sarawak', 'Selangor', 'Terengganu', 'Wilayah Persekutuan Kuala Lumpur', 'Wilayah Persekutuan Labuan', 'Wilayah Persekutuan Putrajaya'
  ];

  async function handleRegister() {
    if (!formData.fullName || !formData.email || !formData.state || !formData.department) {
      error = 'Sila isi semua maklumat';
      return;
    }

    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(formData.email)) {
      error = 'Format email tidak sah';
      return;
    }

    loading = true;
    error = '';

    try {
      const { data: existing } = await supabase
        .from('voters')
        .select('*')
        .eq('email', formData.email)
        .single();

      if (existing) {
        error = 'Email ini sudah didaftarkan! Sila gunakan email lain atau kemaskini maklumat anda.';
        loading = false;
        return;
      }

      const { data, error: dbError } = await supabase
        .from('voters')
        .insert([{
          full_name: formData.fullName,
          email: formData.email,
          state: formData.state,
          department: formData.department,
          has_voted: false,
          registered_at: new Date().toISOString()
        }])
        .select()
        .single();

      if (dbError) throw dbError;

      success = true;
      
      setTimeout(() => {
        goto('/');
      }, 2000);

    } catch (err) {
      error = 'Gagal mendaftar: ' + err.message;
    } finally {
      loading = false;
    }
  }

  function resetForm() {
    formData = {
      fullName: '',
      email: '',
      state: '',
      department: ''
    };
    error = '';
  }
</script>

<div class="page-container">
  <div class="content-wrapper">
    <!-- Header -->
    <div class="header">
      <h1 class="title">Daftar Pengundi</h1>
      <p class="subtitle">Isi maklumat untuk mendaftar sebagai pengundi</p>
    </div>

    {#if success}
      <!-- Success Message -->
      <div class="success-box">
        <div class="success-icon">✓</div>
        <h2 class="success-title">Berjaya Didaftarkan!</h2>
        <p class="success-text">Pendaftaran anda telah berjaya disimpan.</p>
        <p class="success-subtext">Anda boleh kemaskini maklumat anda pada bila-bila masa.</p>
        <div class="redirecting">Mengalihkan ke halaman utama...</div>
      </div>
    {:else}
      <!-- Registration Form -->
      <div class="form-container">
        {#if error}
          <div class="error-box">
            <strong>RALAT:</strong> {error}
          </div>
        {/if}

        <form on:submit|preventDefault={handleRegister}>
          
          <!-- Full Name -->
          <div class="form-group">
            <input
              type="text"
              bind:value={formData.fullName}
              placeholder="Nama Penuh"
              class="form-input"
              required
            />
          </div>

          <!-- Email -->
          <div class="form-group">
            <input
              type="email"
              bind:value={formData.email}
              placeholder="Alamat Email"
              class="form-input"
              required
            />
          </div>

          <!-- State -->
          <div class="form-group">
            <select
              bind:value={formData.state}
              class="form-select"
              required
            >
              <option value="">Negeri Bertugas</option>
              {#each states as state}
                <option value={state}>{state}</option>
              {/each}
            </select>
          </div>

          <!-- Department -->
          <div class="form-group">
            <input
              type="text"
              bind:value={formData.department}
              placeholder="Jabatan"
              class="form-input"
              required
            />
          </div>

          <!-- Buttons -->
          <div class="button-group">
            <button
              type="submit"
              disabled={loading}
              class="btn-primary"
            >
              {loading ? 'Memproses...' : 'Daftar Sekarang'}
            </button>
            
            <button
              type="button"
              on:click={resetForm}
              class="btn-secondary"
            >
              Reset
            </button>
          </div>

          <!-- Info Box -->
          <div class="info-box">
            <p class="info-title">💡 Maklumat Penting</p>
            <p class="info-text">
              Anda boleh kemaskini maklumat ini pada bila-bila masa menggunakan email yang didaftarkan.
            </p>
            <div class="info-footer">
              <p class="info-question">Sudah mendaftar dan ingin kemaskini maklumat?</p>
              <a href="/update" class="info-link">Kemaskini Maklumat Sini →</a>
            </div>
          </div>
        </form>
      </div>
    {/if}

    <!-- Back to Home -->
    <div class="back-link">
      <a href="/">← Kembali ke Halaman Utama</a>
    </div>
  </div>
</div>

<style>
  .page-container {
    min-height: 100vh;
    background: linear-gradient(135deg, #f0fdf4 0%, #d1fae5 50%, #99f6e4 100%);
    padding: 4rem 1rem;
  }

  .content-wrapper {
    max-width: 1200px;
    margin: 0 auto;
  }

  /* Header */
  .header {
    text-align: center;
    margin-bottom: 4rem;
  }

  .title {
    font-size: 5rem;
    font-weight: bold;
    color: #064e3b;
    margin-bottom: 1.5rem;
  }

  .subtitle {
    font-size: 1.75rem;
    color: #4b5563;
  }

  /* Form Container */
  .form-container {
    background: white;
    border-radius: 2rem;
    padding: 4rem;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
  }

  /* Error Box */
  .error-box {
    background: #fef2f2;
    border-left: 8px solid #dc2626;
    color: #991b1b;
    padding: 2rem 2.5rem;
    border-radius: 1rem;
    margin-bottom: 3rem;
    font-size: 1.5rem;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  }

  /* Form Group */
  .form-group {
    margin-bottom: 2rem;
  }

  /* Form Inputs */
  .form-input, .form-select {
    width: 100%;
    padding: 1.5rem 2rem;
    font-size: 1.75rem;
    border: 2px solid #d1d5db;
    border-radius: 1.25rem;
    background: white;
    transition: all 0.2s;
  }

  .form-input:focus, .form-select:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 4px rgba(59, 130, 246, 0.1);
  }

  .form-input::placeholder {
    color: #9ca3af;
  }

  .form-select {
    appearance: none;
    background-image: url("data:image/svg+xml,%3csvg xmlns='http://www.w3.org/2000/svg' fill='none' viewBox='0 0 20 20'%3e%3cpath stroke='%239ca3af' stroke-linecap='round' stroke-linejoin='round' stroke-width='2' d='M6 8l4 4 4-4'/%3e%3c/svg%3e");
    background-position: right 1.5rem center;
    background-repeat: no-repeat;
    background-size: 1.5em 1.5em;
    padding-right: 4rem;
    color: #374151;
  }

  .form-select option:first-child {
    color: #9ca3af;
  }

  /* Info Box */
  .info-box {
    background: linear-gradient(135deg, #eff6ff 0%, #dbeafe 100%);
    border-left: 6px solid #3b82f6;
    padding: 2rem;
    border-radius: 1.5rem;
    margin: 2rem 0;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  }

  .info-title {
    font-size: 1.5rem;
    font-weight: bold;
    color: #1e40af;
    margin-bottom: 1rem;
  }

  .info-text {
    font-size: 1.25rem;
    color: #1e3a8a;
    line-height: 1.6;
    margin-bottom: 1.5rem;
  }

  .info-footer {
    border-top: 2px solid #bfdbfe;
    padding-top: 1.5rem;
  }

  .info-question {
    font-size: 1.1rem;
    color: #1e3a8a;
    margin-bottom: 1rem;
  }

  .info-link {
    color: #2563eb;
    font-weight: bold;
    font-size: 1.25rem;
    text-decoration: none;
  }

  .info-link:hover {
    color: #1e40af;
    text-decoration: underline;
  }

  /* Button Group */
  .button-group {
    display: flex;
    flex-direction: column;
    gap: 1.25rem;
    margin-top: 2rem;
  }

  /* Buttons */
  .btn-primary, .btn-secondary {
    width: 100%;
    padding: 1.75rem 2rem;
    font-size: 1.75rem;
    font-weight: bold;
    border: none;
    border-radius: 1.25rem;
    cursor: pointer;
    transition: all 0.2s;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  }

  .btn-primary {
    background: #3b82f6;
    color: white;
  }

  .btn-primary:hover:not(:disabled) {
    background: #2563eb;
    box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
    transform: translateY(-2px);
  }

  .btn-primary:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .btn-secondary {
    background: #9ca3af;
    color: white;
  }

  .btn-secondary:hover {
    background: #6b7280;
    box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
    transform: translateY(-2px);
  }

  /* Success Box */
  .success-box {
    background: white;
    border-radius: 2rem;
    padding: 5rem;
    text-align: center;
    border: 4px solid #86efac;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
  }

  .success-icon {
    font-size: 8rem;
    color: #22c55e;
    margin-bottom: 2rem;
  }

  .success-title {
    font-size: 4rem;
    font-weight: bold;
    color: #16a34a;
    margin-bottom: 2rem;
  }

  .success-text {
    font-size: 2rem;
    color: #4b5563;
    margin-bottom: 1.5rem;
  }

  .success-subtext {
    font-size: 1.5rem;
    color: #6b7280;
    margin-bottom: 3rem;
  }

  .redirecting {
    font-size: 1.5rem;
    font-weight: bold;
    color: #059669;
    animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
  }

  @keyframes pulse {
    0%, 100% {
      opacity: 1;
    }
    50% {
      opacity: .5;
    }
  }

  /* Back Link */
  .back-link {
    text-align: center;
    margin-top: 3rem;
  }

  .back-link a {
    color: #059669;
    font-weight: bold;
    font-size: 1.5rem;
    text-decoration: none;
  }

  .back-link a:hover {
    color: #047857;
    text-decoration: underline;
  }

  /* Responsive */
  @media (max-width: 768px) {
    .title {
      font-size: 3rem;
    }

    .subtitle {
      font-size: 1.25rem;
    }

    .form-container {
      padding: 2rem;
    }

    .form-input, .form-select, .btn-primary, .btn-secondary {
      font-size: 1.25rem;
      padding: 1.25rem 1.5rem;
    }
  }
</style>