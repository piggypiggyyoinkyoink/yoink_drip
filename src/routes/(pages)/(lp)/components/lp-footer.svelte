<script>
  import RadworksLogo from '$lib/components/illustrations/radworks-logo.svelte';
  import SystemStatusBadge from '$lib/components/system-status-badge/system-status-badge.svelte';
  import { INBOUND_LEAD_FORM_URL } from '$lib/constants';
</script>

<footer>
  <div class="column">
    <h5>Drips</h5>
    <a rel="noreferrer" target="_blank" href="https://docs.drips.network">Docs</a>
    <a rel="noreferrer" target="_blank" href="https://github.com/dripsnetwork">Code on GitHub</a>
    <a rel="noreferrer" target="_blank" href="https://github.com/drips-network/app/issues/new"
      >Report a bug</a
    >
    <a rel="noreferrer" target="_blank" href="https://status.drips.network"
      >System status <SystemStatusBadge /></a
    >
  </div>

  <div class="column">
    <h5>Solutions</h5>
    <a href="/solutions/dependency-funding">Dependency Funding</a>
    <a href="/solutions/retro-pgf">RetroPGF</a>
    <a href="/solutions/programmable-cashflow">Programmable Cashflow</a>
    <a href="/solutions/hackathons">Hackathons</a>
    <a href="/solutions/ecosystem-funding">Ecosystem funding</a>
  </div>

  <div class="column">
    <h5>Get in touch</h5>
    <a rel="noreferrer" target="_blank" href={INBOUND_LEAD_FORM_URL}>Contact</a>
    <a rel="noreferrer" target="_blank" href="https://discord.gg/BakDKKDpHF">Discord</a>
    <a rel="noreferrer" target="_blank" href="https://bsky.app/profile/drips.network">Bluesky</a>
    <a rel="noreferrer" target="_blank" href="https://x.com/dripsnetwork">X</a>
  </div>

  <div class="column">
    <h5>Legal</h5>
    <a class="supported-by" href="https://radworks.org/" target="_blank" rel="noreferrer">
      Supported by <div class="radworks-logo"><RadworksLogo /></div>
    </a>
    <a href="/legal/privacy">Privacy Policy</a>
    <a href="/legal/disclaimer">Disclaimer</a>
    <a href="/legal/access">Access</a>
  </div>
</footer>

<style>
  footer {
    width: 100%;
    margin-top: 4rem;
    display: grid;
    gap: 2rem;
    grid-template-columns: 1fr;
  }

  @media (min-width: 632px) {
    footer {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  @media (min-width: 1296px) {
    footer {
      grid-template-columns: repeat(4, 1fr);
    }
  }

  .column {
    display: flex;
    gap: 0.5rem;
    flex-direction: column;
  }

  a {
    color: var(--color-foreground-level-6);
  }

  .supported-by {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .supported-by > .radworks-logo {
    height: 1.4rem;
    transform: translateY(0.15rem);
  }
</style>
