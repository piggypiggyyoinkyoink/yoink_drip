<script lang="ts">
  import { fade } from 'svelte/transition';

  import ensStore from '$lib/stores/ens';
  import Avatar from '$lib/components/user-avatar/user-avatar.svelte';
  import formatAddress from '$lib/utils/format-address';
  import Tooltip from '../tooltip/tooltip.svelte';

  export let address: string;
  export let showIdentity = true;
  export let showAvatar = true;
  export let hideAvatarOnMobile = false;
  export let disableLink = false;
  export let disableSelection = false;
  export let size: 'small' | 'normal' | 'medium' | 'big' | 'huge' | 'gigantic' = 'normal';
  export let disableTooltip = false;
  export let linkToNewTab = false;
  export let showFullAddress = false;
  export let muted = false;

  export let avatarImgElem: HTMLImageElement | undefined = undefined;
  export let isReverse = false;
  export let tag: string | undefined = undefined;

  $: ensStore.lookup(address);
  $: ens = $ensStore[address];

  $: blockyUrl = `/api/blockies/${address}`;

  $: link = disableLink ? undefined : `/app/${ens?.name ?? address}`;

  $: toDisplay = ens?.name ?? (showFullAddress ? address : formatAddress(address));

  const sizes = {
    small: 16,
    normal: 24,
    medium: 32,
    big: 48,
    huge: 64,
    gigantic: 128,
  };
  $: currentSize = sizes[size];

  const fontClassesEns = {
    small: 'typo-text-small',
    normal: 'typo-text',
    medium: 'typo-text',
    big: 'typo-header-4',
    huge: 'typo-header-3',
    gigantic: 'typo-header-1',
  };
  $: currentFontClassEns = fontClassesEns[size];

  const fontClassesAddress = {
    tiny: 'typo-text-small',
    small: 'typo-text-small',
    normal: 'typo-text',
    medium: 'typo-text',
    big: 'typo-header-4',
    huge: 'typo-header-3',
    gigantic: 'typo-header-1',
  };
  $: currentFontClassAddress = fontClassesAddress[size];

  $: currentFontClass = ens?.name ? currentFontClassEns : currentFontClassAddress;
</script>

<Tooltip text={address} copyable disabled={disableTooltip}>
  <svelte:element
    this={link ? 'a' : 'span'}
    href={link}
    target={linkToNewTab ? '_blank' : undefined}
    class="identity-badge flex items-center relative text-left text-foreground tabular-nums {link &&
    showAvatar &&
    !showIdentity
      ? 'focus-visible:ring-8 focus-visible:ring-primary-level-1 rounded-full mouse:hover:ring-4 mouse:hover:ring-primary-level-1'
      : ''}"
    class:flex-row-reverse={isReverse}
    class:select-none={disableSelection}
    style:height={showAvatar ? `${currentSize}px` : ''}
    style:gap={showAvatar && showIdentity ? `${currentSize / 4}px` : ''}
    class:muted
  >
    {#if showAvatar}
      {#key ens?.avatarUrl + blockyUrl}
        <Avatar
          size={currentSize}
          bind:imgElem={avatarImgElem}
          src={ens?.avatarUrl}
          placeholderSrc={blockyUrl}
        />
      {/key}
    {/if}
    {#if showIdentity}
      <div class="identity relative flex-1 min-w-0">
        <div
          class={`${currentFontClass} identity opacity-0 pointer-events-none flex items-center`}
          class:hideOnMobile={hideAvatarOnMobile}
        >
          {toDisplay}
          {#if ens?.name && showFullAddress}
            <div class="leading-none truncate">{address}</div>
          {/if}
        </div>
        {#key toDisplay}
          <div
            transition:fade={{ duration: 300 }}
            class:text-foreground={size === 'gigantic'}
            class={`${currentFontClass} identity absolute overlay flex items-center`}
            class:hideOnMobile={hideAvatarOnMobile}
          >
            <div class="flex-1 min-w-0">
              <div class="truncate">{toDisplay}</div>
              {#if ens?.name && showFullAddress}
                <div class="leading-none truncate text-foreground-level-5">
                  {address}
                </div>
              {/if}
            </div>
          </div>
        {/key}
      </div>
      {#if tag}
        <div class="tag typo-text-small">
          {tag}
        </div>
      {/if}
    {/if}
  </svelte:element>
  <svelte:fragment slot="tooltip-content">
    {address}
  </svelte:fragment>
</Tooltip>

<style>
  .identity-badge:focus {
    outline: none;
  }

  .identity-badge > .identity {
    transition: background-color 0.3s;
  }

  .identity-badge:focus > .identity {
    background-color: var(--color-primary-level-1);
    border-radius: 0.25rem;
  }

  .mono {
    font-family: var(--typeface-regular);
    white-space: nowrap;
    font-style: normal;
  }

  .muted {
    color: var(--color-foreground-level-6);
  }

  .tag {
    background-color: var(--color-primary-level-1);
    color: var(--color-primary-level-6);
    border-radius: 1rem 0 1rem 1rem;
    padding: 0.125rem 0.5rem;
  }

  @media (max-width: 768px) {
    .hideOnMobile {
      display: none;
    }
  }
</style>
