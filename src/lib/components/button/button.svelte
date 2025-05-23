<script lang="ts">
  import getContrastColor from '$lib/utils/get-contrast-text-color';
  import type { ComponentType } from 'svelte';
  import Spinner from '../spinner/spinner.svelte';
  import { fade } from 'svelte/transition';

  export let variant: 'normal' | 'primary' | 'destructive' | 'destructive-outline' | 'ghost' =
    'normal';
  export let icon: ComponentType | undefined = undefined;
  export let disabled = false;
  export let ariaLabel: string | undefined = undefined;
  export let size: 'small' | 'normal' | 'large' = 'normal';
  export let loading = false;
  export let dataTestId: string | undefined = undefined;
  export let href: string | undefined = undefined;
  export let target: string | undefined = undefined;
  export let rel: string | undefined = undefined;
  export let type: 'submit' | 'reset' | 'button' = 'button';
  export let form: string | undefined = undefined;
  export let justify: 'left' | 'right' | 'center' = 'center';

  $: isDisabled = disabled || loading;

  let el: HTMLButtonElement | HTMLAnchorElement;

  $: primaryColor = el ? getComputedStyle(el).getPropertyValue('--color-primary') : undefined;

  let textColor = 'var(--color-foreground)';
  $: {
    if (variant === 'destructive-outline') {
      textColor = 'var(--color-negative-level-6)';
    } else if (primaryColor && (variant === 'destructive' || variant === 'primary')) {
      textColor = getContrastColor(primaryColor);
    } else {
      textColor = 'var(--color-foreground)';
    }
  }
</script>

<svelte:element
  this={href ? 'a' : 'button'}
  bind:this={el}
  aria-label={ariaLabel}
  {href}
  {target}
  {rel}
  {form}
  class="button size-{size} justify-{justify}"
  class:disabled={isDisabled}
  class:loading
  disabled={isDisabled}
  aria-disabled={isDisabled}
  on:click
  data-testid={dataTestId}
  on:mouseenter|stopPropagation
  on:mouseleave|stopPropagation
  on:focus
  role={href ? 'link' : 'button'}
  style:--color-foreground={variant === 'destructive-outline' ? 'var(--color-negative)' : null}
  type={href ? null : type}
>
  <div
    class:with-icon-text={Boolean(icon) && Boolean($$slots.default)}
    class:with-text={Boolean($$slots.default) && !icon}
    class="inner typo-text {variant}"
    style:color={textColor}
  >
    {#if icon}
      <svelte:component
        this={icon}
        style={variant === 'destructive' || variant === 'primary'
          ? `fill: ${textColor}; transition: fill 0.3s;`
          : 'fill: var(--color-foreground); transition: fill 0.3s;'}
      />
    {/if}
    <slot />
    {#if loading}
      <div out:fade={{ duration: 300 }} class="loading">
        <Spinner />
      </div>
    {/if}
  </div>
</svelte:element>

<style>
  .button {
    display: inline-block;
    height: calc(2rem + 10px);
    min-width: calc(2rem + 4px); /* so just icons are square (w=h) */
    padding: 5px 2px;
    margin: -4px 0;
    transition: opacity 0.3s;
    flex-shrink: 0;
  }

  .button.size-large {
    height: calc(3rem + 10px);
    min-width: calc(3rem + 4px);
  }

  .button.size-small {
    height: 2.5rem;
    min-width: calc(3rem);
  }

  .button .inner {
    height: 100%;
    border-radius: 1rem 0 1rem 1rem;
    display: flex;
    gap: 4px;
    align-items: center;
    justify-content: center;
    white-space: nowrap;
    color: var(--color-foreground);
    user-select: none;
    transition:
      background-color 0.3s,
      color 0.3s,
      transform 0.2s,
      box-shadow 0.2s,
      opacity 0.3s;
    position: relative;
  }

  .button .inner .loading {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    border-radius: 1rem 0 1rem 1rem;
  }

  .button.size-large .inner {
    border-radius: 1.5rem 0 1.5rem 1.5rem;
  }

  .button.size-small .inner {
    font-size: 14px;
  }

  .button .inner.normal {
    background-color: var(--color-background);
  }

  .button .inner:not(.ghost) {
    box-shadow: 0px 0px 0px 1px var(--color-foreground);
  }

  .button:not(.loading) .inner.primary {
    background-color: var(--color-primary);
  }

  .button .inner.destructive {
    background-color: var(--color-negative);
  }

  .button .inner.with-icon-text {
    padding: 0 0.75rem 0 0.5rem;
  }

  .button .inner.with-text {
    padding: 0 0.75rem;
  }

  .button:not(.disabled):hover .inner,
  .button:not(.disabled):focus-visible .inner {
    box-shadow:
      0px 0px 0px 1px var(--color-foreground),
      0 2px 0px 1px var(--color-foreground),
      inset 0 0px 0px 0px var(--color-foreground);
    transform: translateY(-2px);
  }

  .button:not(.disabled):active .inner {
    transform: translateY(0px);
    box-shadow:
      0px 0px 0px 1px var(--color-foreground),
      0 0px 0px 0px var(--color-foreground);
  }

  .button:focus {
    outline: none;
  }

  .button:focus-visible .inner {
    box-shadow: var(--elevation-low);
  }

  .button:focus-visible .inner.normal {
    background-color: var(--color-foreground-level-1);
  }
  .button.disabled {
    opacity: 0.5;
    pointer-events: none;
  }

  .button.justify-left .inner {
    justify-content: start;
  }

  .button.justify-right .inner {
    justify-content: end;
  }
</style>
