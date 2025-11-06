<script>
  import { browser } from '$app/environment';
  import { page } from '$app/stores';
  import { goto } from '$app/navigation';
  
  let isAdmin = false;
  
  $: if (browser) {
    isAdmin = sessionStorage.getItem('adminLoggedIn') === 'true';
  }
  
  $: hideLayout = $page.url.pathname === '/admin/login';
  
  function handleLogout() {
    if (browser) {
      sessionStorage.removeItem('adminLoggedIn');
      sessionStorage.removeItem('adminEmail');
    }
    goto('/');
  }
</script>

<svelte:head>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: Arial, sans-serif; }
  </style>
</svelte:head>

{#if !hideLayout}
<div class="container">
  <header class="header">
    <h1>{isAdmin ? 'Admin e-Voting' : 'e-VOTING PPSM'}</h1>
  </header>

  <div class="main-wrapper">
    <aside class="sidebar">
      <nav>
        {#if isAdmin}
          <a href="/admin/dashboard">Kemaskini Pengundi</a>
          <a href="/admin/candidates">Calon</a>
          <a href="/admin/voting-period">Masa Pengundian</a>
          <button on:click={handleLogout} class="logout">Log Keluar</button>
        {:else}
          <a href="/">home</a>
          <a href="/register">Daftar Pengundi</a>
          <a href="/candidates">Senarai Pencalonan</a>
          <a href="/nominate">Hantar Pencalonan</a>
          <a href="/vote">Pengundian</a>
          <a href="/keputusan">Keputusan</a>
          <a href="/admin/login">Admin</a>
        {/if}
      </nav>
    </aside>

    <main class="content">
      <slot />
    </main>
  </div>
</div>
{:else}
  <slot />
{/if}

<style>
  .container {
    display: flex;
    flex-direction: column;
    min-height: 100vh;
  }
  
  .header {
    height: 12rem;
    background: linear-gradient(to right, #0f766e, #0891b2, #1e40af);
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .header h1 {
    font-size: 3.75rem;
    font-weight: bold;
    color: white;
    text-shadow: 3px 3px 6px rgba(0,0,0,0.5);
    letter-spacing: 0.1em;
  }
  
  .main-wrapper {
    display: flex;
    flex: 1;
  }
  
  .sidebar {
    width: 18rem;
    background: linear-gradient(to bottom, #22d3ee, #06b6d4);
  }
  
  .sidebar a, .sidebar button {
    display: block;
    padding: 1.25rem 2rem;
    font-size: 1.25rem;
    font-weight: 600;
    color: #111827;
    border-bottom: 1px solid #0891b2;
    text-decoration: none;
    transition: background 0.3s;
    border: none;
    width: 100%;
    text-align: left;
    background: transparent;
    cursor: pointer;
  }
  
  .sidebar a:hover, .sidebar button:hover {
    background-color: #67e8f9;
  }
  
  .sidebar .logout {
    color: #b91c1c;
  }
  
  .content {
    flex: 1;
    background: white;
    padding: 2rem;
    overflow-y: auto;
  }
</style>