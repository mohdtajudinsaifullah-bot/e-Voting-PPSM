<script>
  // No script needed for this page
</script>

<div style="padding: 3rem;">
  
  <!-- Logo Animated -->
  <div style="text-align: center; margin-bottom: 2rem;">
    <img 
      src="/images/Logo PPSM.png" 
      alt="Logo PPSM" 
      style="width: 150px; animation: pulse 2s ease-in-out infinite;"
    />
  </div>

  <!-- Welcome Text (Centered & Bigger) -->
  <div style="text-align: center; max-width: 800px; margin: 0 auto 3rem auto;">
    <h2 style="font-size: 3.5rem; font-weight: bold; margin-bottom: 1.5rem; color: #111827;">
      Selamat Datang
    </h2>
    <p style="font-size: 1.5rem; line-height: 1.75; color: #6b7280;">
      Sistem E-Voting Persatuan Pegawai Syariah Malaysia (PPSM). 
      Sila gunakan menu di sebelah kiri untuk tindakan atau capaian maklumat.
    </p>
  </div>

  <!-- SEAMLESS Scrolling Announcement -->
  <div class="announcement-container">
    <div class="announcement-scroll">
      <span class="announcement-content">
        🗳️ Pengundian akan bermula pada 15 Disember 2024&nbsp;&nbsp;|&nbsp;&nbsp;
        📢 Pastikan anda mendaftar sebagai pengundi&nbsp;&nbsp;|&nbsp;&nbsp;
        ✅ Hanya pengundi berdaftar sahaja yang boleh mengundi&nbsp;&nbsp;|&nbsp;&nbsp;
        🎯 Sila pastikan maklumat anda adalah betul&nbsp;&nbsp;|&nbsp;&nbsp;
      </span>
      <span class="announcement-content">
        🗳️ Pengundian akan bermula pada 15 Disember 2024&nbsp;&nbsp;|&nbsp;&nbsp;
        📢 Pastikan anda mendaftar sebagai pengundi&nbsp;&nbsp;|&nbsp;&nbsp;
        ✅ Hanya pengundi berdaftar sahaja yang boleh mengundi&nbsp;&nbsp;|&nbsp;&nbsp;
        🎯 Sila pastikan maklumat anda adalah betul&nbsp;&nbsp;|&nbsp;&nbsp;
      </span>
    </div>
  </div>

</div>

<style>
  /* Logo pulse animation */
  @keyframes pulse {
    0%, 100% { 
      opacity: 1; 
      transform: scale(1); 
    }
    50% { 
      opacity: 0.8; 
      transform: scale(1.05); 
    }
  }
  
  /* Seamless scrolling animation */
  @keyframes seamlessScroll {
    0% { 
      transform: translateX(0%); 
    }
    100% { 
      transform: translateX(-50%); 
    }
  }

  /* Announcement container */
  .announcement-container {
    background: linear-gradient(135deg, #0ea5e9 0%, #06b6d4 100%);
    padding: 1rem 0;
    overflow: hidden;
    border-radius: 12px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    position: relative;
  }

  /* Scrolling wrapper - THE KEY! */
  .announcement-scroll {
    display: inline-flex;
    animation: seamlessScroll 15s linear infinite;
    white-space: nowrap;
    will-change: transform;
  }

  /* Content styling */
  .announcement-content {
    font-size: 1.25rem;
    font-weight: 600;
    color: white;
    display: inline-block;
    padding-right: 0;
  }

  /* Pause on hover */
  .announcement-container:hover .announcement-scroll {
    animation-play-state: paused;
  }

  /* Responsive */
  @media (max-width: 768px) {
    .announcement-content {
      font-size: 1rem;
    }
  }
</style>