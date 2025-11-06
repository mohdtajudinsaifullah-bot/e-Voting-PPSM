<!-- src/routes/admin/dashboard/+layout.svelte -->
<script>
  import { page } from '$app/stores';
  import { goto } from '$app/navigation';
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let adminSession = null;

  const menuItems = [
    { 
      path: '/admin/dashboard', 
      label: 'Dashboard', 
      icon: '📊' 
    },
    { 
      path: '/admin/voting-time', 
      label: 'Tetapkan Masa', 
      icon: '🕐' 
    },
    { 
      path: '/admin/voters', 
      label: 'Kemaskini Pengundi', 
      icon: '👥' 
    },
    { 
      path: '/admin/candidates', 
      label: 'Masukkan Calon', 
      icon: '🎯' 
    }
  ];

  onMount(() => {
    if (browser) {
      const session = localStorage.getItem('admin_session');
      if (!session) {
        goto('/admin');
        return;
      }
      adminSession = JSON.parse(session);
    }
  });

  function logout() {
    if (browser) {
      localStorage.removeItem('admin_session');
    }
    goto('/admin');
  }

  function isActive(itemPath) {
    return $page.url.pathname === itemPath;
  }
</script>

<div class="flex h-screen bg-gray-100">
  <!-- Sidebar -->
  <aside class="w-64 bg-white shadow-lg flex flex-col">
    <!-- Logo/Brand -->
    <div class="p-6 border-b border-gray-200">
      <h1 class="text-xl font-bold text-gray-800">Admin Panel</h1>
      <p class="text-sm text-gray-600 mt-1">E-Voting System</p>
    </div>

    <!-- User Info -->
    <div class="px-6 py-4 border-b border-gray-200">
      <div class="flex items-center space-x-3">
        <div class="w-10 h-10 bg-indigo-100 rounded-full flex items-center justify-center">
          <span class="text-indigo-600 font-semibold text-lg">
            {adminSession?.name?.charAt(0) || 'A'}
          </span>
        </div>
        <div class="flex-1 min-w-0">
          <p class="text-sm font-semibold text-gray-800 truncate">
            {adminSession?.name || 'Admin'}
          </p>
          <p class="text-xs text-gray-500 truncate">
            {adminSession?.email || ''}
          </p>
        </div>
      </div>
    </div>

    <!-- Navigation Menu -->
    <nav class="flex-1 px-3 py-4 overflow-y-auto">
      {#each menuItems as item}
        <a
          href={item.path}
          class="flex items-center px-4 py-3 mb-1 rounded-lg transition {
            isActive(item.path)
              ? 'bg-indigo-50 text-indigo-700 font-semibold'
              : 'text-gray-700 hover:bg-gray-50'
          }"
        >
          <span class="text-xl mr-3">{item.icon}</span>
          <span class="text-sm">{item.label}</span>
        </a>
      {/each}
    </nav>

    <!-- Logout Button -->
    <div class="p-4 border-t border-gray-200">
      <button
        on:click={logout}
        class="w-full flex items-center justify-center px-4 py-3 bg-red-500 hover:bg-red-600 text-white rounded-lg transition font-medium"
      >
        <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 16l4-4m0 0l-4-4m4 4H7m6 4v1a3 3 0 01-3 3H6a3 3 0 01-3-3V7a3 3 0 013-3h4a3 3 0 013 3v1"/>
        </svg>
        Log Keluar
      </button>
    </div>
  </aside>

  <!-- Main Content Area -->
  <main class="flex-1 overflow-y-auto">
    <!-- Top Header -->
    <header class="bg-white shadow-sm px-8 py-4 border-b border-gray-200">
      <div class="flex items-center justify-between">
        <div>
          <h2 class="text-2xl font-bold text-gray-800">
            {menuItems.find(item => isActive(item.path))?.label || 'Dashboard'}
          </h2>
          <p class="text-sm text-gray-600 mt-1">
            {new Date().toLocaleDateString('ms-MY', { dateStyle: 'full' })}
          </p>
        </div>
      </div>
    </header>

    <!-- Page Content -->
    <div class="p-8">
      <slot />
    </div>
  </main>
</div>

<style>
  :global(body) {
    margin: 0;
    overflow: hidden;
  }
</style>