<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { page } from '$app/stores';

  let adminUser = null;
  let currentPath = '';

  onMount(() => {
    const stored = localStorage.getItem('admin_user');
    if (!stored) {
      goto('/admin/login');
      return;
    }
    adminUser = JSON.parse(stored);
    currentPath = window.location.pathname;
  });

  function logout() {
    if (confirm('Adakah anda pasti mahu log keluar?')) {
      localStorage.removeItem('admin_user');
      goto('/admin/login');
    }
  }

  const menuItems = [
    { path: '/admin/pengundi', label: 'Kemaskini Pengundi', icon: '👥' },
    { path: '/admin/calon', label: 'Calon', icon: '🎯' },
    { path: '/admin/tetapan-calon', label: 'Tetapan Calon Undian', icon: '⚙️' },
    { path: '/admin/masa-pengundian', label: 'Masa Pengundian', icon: '📅' },
  ];

  function isActive(path) {
    return currentPath === path;
  }
</script>

<svelte:head>
  <title>Admin Dashboard - e-VOTING PPSM</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
      overflow-y: auto !important;
    }
  </style>
</svelte:head>

<div style="min-height: 100vh; background: #f5f5f5;">
  
  <!-- Header -->
  <div style="
    background: linear-gradient(135deg, rgba(14, 116, 144, 0.95), rgba(6, 95, 70, 0.95));
    padding: 3rem 2rem;
    color: white;
    box-shadow: 0 4px 6px rgba(0,0,0,0.1);
    text-align: center;
  ">
    <h1 style="font-size: 3rem; font-weight: 700; text-shadow: 2px 2px 4px rgba(0,0,0,0.3); letter-spacing: 0.05em;">
      Admin e-Voting
    </h1>
  </div>

  <!-- Horizontal Menu Bar -->
  <div style="
    background: linear-gradient(90deg, #0e7490 0%, #14b8a6 50%, #0d9488 100%);
    box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  ">
    <div style="max-width: 1400px; margin: 0 auto; display: flex; align-items: center; justify-content: space-between;">
      
      <!-- Menu Items -->
      <div style="display: flex;">
        {#each menuItems as item}
          <a
            href={item.path}
            style="
              display: flex;
              align-items: center;
              gap: 0.75rem;
              padding: 1.25rem 2rem;
              color: white;
              text-decoration: none;
              font-size: 1.125rem;
              font-weight: 600;
              transition: all 0.2s;
              border-bottom: 4px solid transparent;
              {isActive(item.path) ? 'background: rgba(255,255,255,0.2); border-bottom-color: white;' : ''}
            "
            on:mouseenter={(e) => {
              if (!isActive(item.path)) {
                e.currentTarget.style.background = 'rgba(255,255,255,0.1)';
              }
            }}
            on:mouseleave={(e) => {
              if (!isActive(item.path)) {
                e.currentTarget.style.background = 'transparent';
              }
            }}
          >
            <span style="font-size: 1.5rem;">{item.icon}</span>
            <span>{item.label}</span>
          </a>
        {/each}
      </div>

      <!-- Logout Button -->
      <button
        on:click={logout}
        style="
          display: flex;
          align-items: center;
          gap: 0.75rem;
          padding: 1.25rem 2rem;
          margin-right: 1rem;
          color: white;
          background: rgba(239, 68, 68, 0.2);
          border: 2px solid rgba(255,255,255,0.3);
          border-radius: 12px;
          font-size: 1.125rem;
          font-weight: 600;
          cursor: pointer;
          transition: all 0.2s;
          font-family: inherit;
        "
        on:mouseenter={(e) => {
          e.currentTarget.style.background = 'rgba(239, 68, 68, 0.4)';
          e.currentTarget.style.borderColor = 'white';
        }}
        on:mouseleave={(e) => {
          e.currentTarget.style.background = 'rgba(239, 68, 68, 0.2)';
          e.currentTarget.style.borderColor = 'rgba(255,255,255,0.3)';
        }}
      >
        <span style="font-size: 1.5rem;">🚪</span>
        <span>Log Keluar</span>
      </button>
    </div>
  </div>

  <!-- Main Content Area -->
  <div style="max-width: 1400px; margin: 0 auto; padding: 2rem;">
    <slot />
  </div>

</div>

<style>
  button {
    font-family: inherit;
  }
</style>