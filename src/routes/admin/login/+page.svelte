<script>
  import { goto } from '$app/navigation';
  import { supabase } from '$lib/supabaseClient';
  
  let email = '';
  let password = '';
  let error = '';
  let loading = false;
  let debugInfo = ''; // For debugging

  async function handleLogin() {
    if (!email || !password) {
      error = 'Sila masukkan email dan password';
      return;
    }

    loading = true;
    error = '';
    debugInfo = ''; // Clear debug info

    try {
      console.log('=== LOGIN ATTEMPT ===');
      console.log('Email entered:', email);
      console.log('Password length:', password.length);

      // Check against admins table with EXACT match
      const { data: admins, error: dbError, count } = await supabase
        .from('admins')
        .select('*', { count: 'exact' })
        .eq('email', email)
        .eq('password', password);

      console.log('Query result:', { admins, dbError, count });
      console.log('Data returned:', admins);

      if (dbError) {
        console.error('Database error:', dbError);
        error = 'Ralat database: ' + dbError.message;
        debugInfo = `DB Error: ${dbError.message}`;
        loading = false;
        return;
      }

      // Check if we got results
      if (!admins || admins.length === 0) {
        console.log('No matching admin found');
        
        // Try to get all admins for debugging
        const { data: allAdmins } = await supabase
          .from('admins')
          .select('email, password');
        
        console.log('All admins in DB:', allAdmins);
        
        error = 'Email atau password tidak sah';
        debugInfo = `No match found. Total admins: ${allAdmins?.length || 0}`;
        loading = false;
        return;
      }

      // Success! Store admin session
      const admin = admins[0];
      console.log('✅ Login successful!', admin);
      
      localStorage.setItem('admin_user', JSON.stringify({
        id: admin.id,
        email: admin.email,
        name: admin.name || 'Admin'
      }));

      // Redirect to admin dashboard
      goto('/admin/');

    } catch (err) {
      console.error('Login error:', err);
      error = 'Gagal log masuk: ' + err.message;
      debugInfo = `Exception: ${err.message}`;
    } finally {
      loading = false;
    }
  }

  function handleKeyPress(event) {
    if (event.key === 'Enter') {
      handleLogin();
    }
  }
</script>

<svelte:head>
  <title>Admin Login - e-VOTING PPSM</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
  </style>
</svelte:head>

<div style="min-height: 100vh; background: linear-gradient(135deg, #1e3a8a 0%, #3b82f6 50%, #06b6d4 100%); display: flex; align-items: center; justify-content: center; padding: 2rem;">
  
  <div style="width: 100%; max-width: 450px;">
    
    <!-- Header -->
    <div style="text-align: center; margin-bottom: 2rem;">
      <div style="font-size: 3rem; margin-bottom: 0.5rem;">🔐</div>
      <h1 style="font-size: 2.5rem; font-weight: 700; color: white; margin-bottom: 0.5rem; text-shadow: 2px 2px 4px rgba(0,0,0,0.2);">
        Admin Panel
      </h1>
      <p style="color: rgba(255,255,255,0.9); font-size: 1.125rem;">
        Sistem E-Voting Persatuan
      </p>
    </div>

    <!-- Login Card -->
    <div style="background: white; border-radius: 24px; box-shadow: 0 20px 60px rgba(0,0,0,0.3); padding: 3rem;">
      
      <h2 style="font-size: 1.75rem; font-weight: 700; color: #111827; margin-bottom: 0.5rem;">
        Log Masuk
      </h2>
      <p style="color: #6b7280; margin-bottom: 2rem;">
        Email Admin
      </p>

      {#if error}
        <div style="background: #fef2f2; border-left: 4px solid #ef4444; color: #991b1b; padding: 1rem 1.25rem; border-radius: 12px; margin-bottom: 1.5rem;">
          ⚠️ {error}
          {#if debugInfo}
            <div style="font-size: 0.75rem; margin-top: 0.5rem; opacity: 0.7;">
              Debug: {debugInfo}
            </div>
          {/if}
        </div>
      {/if}

      <form on:submit|preventDefault={handleLogin}>
        
        <!-- Email -->
        <div style="margin-bottom: 1.5rem;">
          <label style="display: block; font-size: 0.875rem; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
            Email
          </label>
          <input
            type="email"
            bind:value={email}
            on:keypress={handleKeyPress}
            placeholder="admin@example.com"
            required
            style="width: 100%; padding: 0.875rem 1rem; font-size: 1rem; border: 2px solid #d1d5db; border-radius: 12px; transition: all 0.2s; font-family: inherit;"
            on:focus={(e) => {
              e.target.style.borderColor = '#3b82f6';
              e.target.style.outline = 'none';
              e.target.style.boxShadow = '0 0 0 3px rgba(59, 130, 246, 0.1)';
            }}
            on:blur={(e) => {
              e.target.style.borderColor = '#d1d5db';
              e.target.style.boxShadow = 'none';
            }}
          />
        </div>

        <!-- Password -->
        <div style="margin-bottom: 2rem;">
          <label style="display: block; font-size: 0.875rem; font-weight: 600; color: #374151; margin-bottom: 0.5rem;">
            Password
          </label>
          <input
            type="password"
            bind:value={password}
            on:keypress={handleKeyPress}
            placeholder="••••••••"
            required
            style="width: 100%; padding: 0.875rem 1rem; font-size: 1rem; border: 2px solid #d1d5db; border-radius: 12px; transition: all 0.2s; font-family: inherit;"
            on:focus={(e) => {
              e.target.style.borderColor = '#3b82f6';
              e.target.style.outline = 'none';
              e.target.style.boxShadow = '0 0 0 3px rgba(59, 130, 246, 0.1)';
            }}
            on:blur={(e) => {
              e.target.style.borderColor = '#d1d5db';
              e.target.style.boxShadow = 'none';
            }}
          />
        </div>

        <!-- Login Button -->
        <button
          type="submit"
          disabled={loading}
          style="width: 100%; background: linear-gradient(135deg, #1e3a8a, #3b82f6); color: white; padding: 1rem; border-radius: 12px; font-size: 1.125rem; font-weight: 700; border: none; cursor: pointer; transition: all 0.2s; font-family: inherit; {loading ? 'opacity: 0.6; cursor: not-allowed;' : ''}"
          on:mouseenter={(e) => !loading && (e.target.style.transform = 'translateY(-2px)', e.target.style.boxShadow = '0 6px 20px rgba(59, 130, 246, 0.4)')}
          on:mouseleave={(e) => !loading && (e.target.style.transform = 'translateY(0)', e.target.style.boxShadow = 'none')}
        >
          {loading ? '⏳ Logging in...' : '🔐 Log Masuk'}
        </button>

      </form>

      <!-- Info -->
      <div style="margin-top: 2rem; padding-top: 2rem; border-top: 1px solid #e5e7eb; text-align: center;">
        <p style="color: #6b7280; font-size: 0.875rem;">
          Admin sahaja boleh akses panel ini
        </p>
      </div>

      <!-- Debug Info (Development only) -->
      {#if import.meta.env.DEV}
        <div style="margin-top: 1rem; padding: 1rem; background: #f3f4f6; border-radius: 8px; font-size: 0.75rem; color: #6b7280;">
          <strong>Debug Info:</strong><br>
          Email: {email || '(empty)'}<br>
          Password length: {password.length}<br>
          Table: admins<br>
          <button 
            type="button"
            on:click={() => {
              email = 'm_tajudin@esyariah.gov.my';
              password = 'T@jud!n321';
            }}
            style="margin-top: 0.5rem; padding: 0.25rem 0.5rem; background: #3b82f6; color: white; border: none; border-radius: 4px; cursor: pointer; font-family: inherit; font-size: 0.75rem;"
          >
            Fill Test Credentials
          </button>
        </div>
      {/if}

    </div>

    <!-- Back to Home -->
    <div style="text-align: center; margin-top: 2rem;">
      <a href="/" style="color: white; font-weight: 600; text-decoration: none; font-size: 1rem; transition: opacity 0.2s;">
        ← Kembali ke Halaman Utama
      </a>
    </div>

  </div>

</div>

<style>
  input:focus {
    outline: none;
  }
</style>