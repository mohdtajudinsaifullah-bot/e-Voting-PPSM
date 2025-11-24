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
      Sistem E-Voting Persatuan Penolong Pegawai Syariah Malaysia (PPSM). 
      Sila gunakan menu di sebelah kiri untuk navigasi.
    </p>
  </div>

  <!-- Scrolling Announcement (IMPROVED) -->
  <div class="announcement-container">
    <div class="announcement-scroll">
      <span class="announcement-text">
        🗳️ Pengundian akan bermula pada 15 Disember 2024 | 
        📢 Pastikan anda mendaftar sebagai pengundi | 
        ✅ Hanya pengundi berdaftar sahaja yang boleh mengundi |
        🎯 Sila pastikan maklumat anda adalah betul
      </span>
      <!-- Duplicate for seamless loop -->
      <span class="announcement-text">
        🗳️ Pengundian akan bermula pada 15 Disember 2024 | 
        📢 Pastikan anda mendaftar sebagai pengundi | 
        ✅ Hanya pengundi berdaftar sahaja yang boleh mengundi |
        🎯 Sila pastikan maklumat anda adalah betul
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
  
  /* Scrolling text animation */
  @keyframes scroll {
    0% { 
      transform: translateX(0); 
    }
    100% { 
      transform: translateX(-50%); 
    }
  }

  /* Announcement container styling */
  .announcement-container {
    background: linear-gradient(135deg, #0ea5e9 0%, #06b6d4 100%);
    padding: 1rem 0;
    overflow: hidden;
    border-radius: 12px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    position: relative;
  }

  /* Scrolling animation wrapper */
  .announcement-scroll {
    display: flex;
    animation: scroll 20s linear infinite;
    white-space: nowrap;
  }

  /* Announcement text styling */
  .announcement-text {
    font-size: 1.25rem;
    font-weight: 600;
    color: white;
    padding: 0 2rem;
    display: inline-block;
  }

  /* Pause animation on hover */
  .announcement-container:hover .announcement-scroll {
    animation-play-state: paused;
  }

  /* Responsive adjustments */
  @media (max-width: 768px) {
    .announcement-text {
      font-size: 1rem;
    }
  }
</style>