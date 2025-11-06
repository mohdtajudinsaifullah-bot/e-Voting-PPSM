<script>
  import { supabase } from '$lib/supabaseClient';

  let csvText = '';
  let loading = false;
  let error = '';
  let success = false;
  let uploadResults = {
    total: 0,
    success: 0,
    failed: 0,
    errors: []
  };

  let existingNames = [];
  let showExisting = false;

  async function fetchExistingNames() {
    try {
      const { data, error: dbError } = await supabase
        .from('senarai_nama')
        .select('*')
        .order('nama', { ascending: true });

      if (dbError) throw dbError;
      existingNames = data || [];
      showExisting = true;
    } catch (err) {
      console.error('Error fetching names:', err);
    }
  }

  async function handleUpload() {
    if (!csvText.trim()) {
      error = 'Sila masukkan data';
      return;
    }

    loading = true;
    error = '';
    success = false;
    uploadResults = { total: 0, success: 0, failed: 0, errors: [] };

    try {
      // Parse CSV
      const lines = csvText.trim().split('\n');
      const records = [];

      for (let i = 0; i < lines.length; i++) {
        const line = lines[i].trim();
        if (!line) continue;

        // Split by comma or tab
        const parts = line.split(/[,\t]/).map(p => p.trim());
        
        if (parts.length >= 2) {
          records.push({
            nama: parts[0],
            negeri_bertugas: parts[1]
          });
        }
      }

      uploadResults.total = records.length;

      if (records.length === 0) {
        error = 'Tiada data yang sah dijumpai';
        loading = false;
        return;
      }

      // Insert records
      for (const record of records) {
        try {
          const { error: dbError } = await supabase
            .from('senarai_nama')
            .insert([record]);

          if (dbError) {
            uploadResults.failed++;
            uploadResults.errors.push(`${record.nama}: ${dbError.message}`);
          } else {
            uploadResults.success++;
          }
        } catch (err) {
          uploadResults.failed++;
          uploadResults.errors.push(`${record.nama}: ${err.message}`);
        }
      }

      success = true;
      if (uploadResults.failed === 0) {
        csvText = ''; // Clear form if all successful
      }

    } catch (err) {
      error = 'Gagal memproses data: ' + err.message;
    } finally {
      loading = false;
    }
  }

  async function clearAllData() {
    if (!confirm('AMARAN: Ini akan PADAM SEMUA data dalam senarai nama. Adakah anda pasti?')) {
      return;
    }

    loading = true;
    try {
      const { error: dbError } = await supabase
        .from('senarai_nama')
        .delete()
        .neq('id', '00000000-0000-0000-0000-000000000000'); // Delete all

      if (dbError) throw dbError;

      alert('Semua data telah dipadam');
      existingNames = [];
      
    } catch (err) {
      error = 'Gagal memadam data: ' + err.message;
    } finally {
      loading = false;
    }
  }
</script>

<div class="page-container">
  <div class="content-wrapper">
    <!-- Header -->
    <div class="header">
      <h1 class="title">Import Senarai Nama Calon</h1>
      <p class="subtitle">Muat naik senarai nama calon dari Google Sheet atau CSV</p>
    </div>

    <!-- Instructions -->
    <div class="instructions-box">
      <h3>📋 Panduan Upload:</h3>
      <ol>
        <li>Buka Google Sheet anda</li>
        <li>Pilih 2 kolum: <strong>Nama</strong> dan <strong>Negeri Bertugas</strong></li>
        <li>Copy data (termasuk atau tanpa header)</li>
        <li>Paste dalam kotak di bawah</li>
        <li>Klik "Upload Senarai"</li>
      </ol>
      
      <div class="example-box">
        <strong>Contoh Format:</strong>
        <pre>Ahmad bin Ali, Selangor
Fatimah Zahra, Kuala Lumpur
Hassan Mahmud, Johor</pre>
      </div>
    </div>

    <!-- Upload Form -->
    <div class="upload-box">
      {#if error}
        <div class="error-box">
          <strong>RALAT:</strong> {error}
        </div>
      {/if}

      {#if success}
        <div class="success-box">
          <h3>✅ Upload Selesai!</h3>
          <div class="results">
            <div class="result-item">
              <span class="label">Jumlah:</span>
              <span class="value">{uploadResults.total}</span>
            </div>
            <div class="result-item success">
              <span class="label">Berjaya:</span>
              <span class="value">{uploadResults.success}</span>
            </div>
            {#if uploadResults.failed > 0}
              <div class="result-item failed">
                <span class="label">Gagal:</span>
                <span class="value">{uploadResults.failed}</span>
              </div>
            {/if}
          </div>

          {#if uploadResults.errors.length > 0}
            <div class="errors-list">
              <strong>Ralat yang berlaku:</strong>
              <ul>
                {#each uploadResults.errors as err}
                  <li>{err}</li>
                {/each}
              </ul>
            </div>
          {/if}
        </div>
      {/if}

      <form on:submit|preventDefault={handleUpload}>
        <div class="form-group">
          <label for="csvText">Paste Data dari Google Sheet:</label>
          <textarea
            id="csvText"
            bind:value={csvText}
            placeholder="Paste data dari Google Sheet di sini...
Contoh:
Ahmad bin Ali, Selangor
Fatimah Zahra, Kuala Lumpur"
            rows="15"
            class="csv-textarea"
          ></textarea>
          <p class="hint">Format: Nama, Negeri Bertugas (satu baris satu orang)</p>
        </div>

        <div class="button-group">
          <button
            type="submit"
            disabled={loading}
            class="btn-primary"
          >
            {loading ? 'Memuat Naik...' : 'Upload Senarai'}
          </button>
          
          <button
            type="button"
            on:click={() => csvText = ''}
            class="btn-secondary"
          >
            Clear
          </button>
        </div>
      </form>
    </div>

    <!-- View Existing Data -->
    <div class="view-section">
      <button 
        on:click={fetchExistingNames}
        class="btn-view"
      >
        {showExisting ? 'Refresh' : 'Lihat'} Senarai Nama ({existingNames.length})
      </button>

      {#if showExisting && existingNames.length > 0}
        <div class="existing-list">
          <div class="list-header">
            <h3>Senarai Nama Semasa ({existingNames.length})</h3>
            <button 
              on:click={clearAllData}
              class="btn-danger"
            >
              Padam Semua
            </button>
          </div>
          <div class="names-grid">
            {#each existingNames as person}
              <div class="name-card">
                <div class="name">{person.nama}</div>
                <div class="state">{person.negeri_bertugas}</div>
              </div>
            {/each}
          </div>
        </div>
      {/if}
    </div>

    <!-- Back Link -->
    <div class="back-link">
      <a href="/">← Kembali ke Halaman Utama</a>
    </div>
  </div>
</div>

<style>
  .page-container {
    min-height: 100vh;
    background: linear-gradient(135deg, #fef3c7 0%, #fde68a 50%, #fcd34d 100%);
    padding: 4rem 1rem;
  }

  .content-wrapper {
    max-width: 1200px;
    margin: 0 auto;
  }

  .header {
    text-align: center;
    margin-bottom: 3rem;
  }

  .title {
    font-size: 4rem;
    font-weight: bold;
    color: #78350f;
    margin-bottom: 1rem;
  }

  .subtitle {
    font-size: 1.5rem;
    color: #4b5563;
  }

  .instructions-box {
    background: white;
    border-radius: 1.5rem;
    padding: 2rem;
    margin-bottom: 2rem;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
  }

  .instructions-box h3 {
    color: #78350f;
    font-size: 1.5rem;
    margin-bottom: 1rem;
  }

  .instructions-box ol {
    margin-left: 1.5rem;
    margin-bottom: 1.5rem;
  }

  .instructions-box li {
    font-size: 1.1rem;
    color: #4b5563;
    margin-bottom: 0.5rem;
  }

  .example-box {
    background: #fef3c7;
    border-left: 4px solid #f59e0b;
    padding: 1rem;
    border-radius: 0.5rem;
  }

  .example-box pre {
    margin-top: 0.5rem;
    font-size: 1rem;
    color: #78350f;
  }

  .upload-box {
    background: white;
    border-radius: 2rem;
    padding: 3rem;
    box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1);
    margin-bottom: 2rem;
  }

  .error-box {
    background: #fef2f2;
    border-left: 8px solid #dc2626;
    color: #991b1b;
    padding: 1.5rem;
    border-radius: 0.75rem;
    margin-bottom: 2rem;
    font-size: 1.2rem;
  }

  .success-box {
    background: #f0fdf4;
    border-left: 8px solid #16a34a;
    color: #166534;
    padding: 1.5rem;
    border-radius: 0.75rem;
    margin-bottom: 2rem;
  }

  .success-box h3 {
    margin-bottom: 1rem;
    font-size: 1.5rem;
  }

  .results {
    display: flex;
    gap: 2rem;
    margin-bottom: 1rem;
  }

  .result-item {
    display: flex;
    gap: 0.5rem;
    align-items: center;
  }

  .result-item .label {
    font-weight: 600;
  }

  .result-item .value {
    font-size: 1.5rem;
    font-weight: bold;
  }

  .result-item.success .value {
    color: #16a34a;
  }

  .result-item.failed .value {
    color: #dc2626;
  }

  .errors-list {
    margin-top: 1rem;
    padding: 1rem;
    background: #fee2e2;
    border-radius: 0.5rem;
  }

  .errors-list ul {
    margin-top: 0.5rem;
    margin-left: 1.5rem;
  }

  .form-group {
    margin-bottom: 1.5rem;
  }

  .form-group label {
    display: block;
    font-size: 1.2rem;
    font-weight: bold;
    color: #78350f;
    margin-bottom: 0.75rem;
  }

  .csv-textarea {
    width: 100%;
    padding: 1rem;
    font-size: 1rem;
    font-family: 'Courier New', monospace;
    border: 2px solid #d1d5db;
    border-radius: 0.75rem;
    resize: vertical;
  }

  .csv-textarea:focus {
    outline: none;
    border-color: #f59e0b;
    box-shadow: 0 0 0 3px rgba(245, 158, 11, 0.1);
  }

  .hint {
    font-size: 0.9rem;
    color: #6b7280;
    margin-top: 0.5rem;
  }

  .button-group {
    display: flex;
    gap: 1rem;
  }

  .btn-primary, .btn-secondary, .btn-view, .btn-danger {
    padding: 1rem 2rem;
    border-radius: 0.75rem;
    font-size: 1.2rem;
    font-weight: bold;
    border: none;
    cursor: pointer;
    transition: all 0.2s;
  }

  .btn-primary {
    background: #f59e0b;
    color: white;
    flex: 1;
  }

  .btn-primary:hover:not(:disabled) {
    background: #d97706;
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
  }

  .view-section {
    margin-top: 3rem;
  }

  .btn-view {
    background: #0891b2;
    color: white;
    width: 100%;
  }

  .btn-view:hover {
    background: #0e7490;
  }

  .existing-list {
    background: white;
    border-radius: 2rem;
    padding: 2rem;
    margin-top: 1rem;
    box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1);
  }

  .list-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
    padding-bottom: 1rem;
    border-bottom: 2px solid #fde68a;
  }

  .list-header h3 {
    font-size: 1.75rem;
    color: #78350f;
  }

  .btn-danger {
    background: #dc2626;
    color: white;
  }

  .btn-danger:hover {
    background: #b91c1c;
  }

  .names-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 1rem;
  }

  .name-card {
    background: #fef3c7;
    padding: 1rem;
    border-radius: 0.75rem;
    border-left: 4px solid #f59e0b;
  }

  .name-card .name {
    font-size: 1.1rem;
    font-weight: 600;
    color: #78350f;
    margin-bottom: 0.25rem;
  }

  .name-card .state {
    font-size: 0.9rem;
    color: #6b7280;
  }

  .back-link {
    text-align: center;
    margin-top: 3rem;
  }

  .back-link a {
    color: #d97706;
    font-weight: bold;
    font-size: 1.25rem;
    text-decoration: none;
  }

  .back-link a:hover {
    text-decoration: underline;
  }
</style>