<script lang="ts">
  import Button from '$lib/components/button/button.svelte';
  import FormField from '$lib/components/form-field/form-field.svelte';
  import ArrowLeft from '$lib/components/icons/ArrowLeft.svelte';
  import StandaloneFlowStepLayout from '$lib/components/standalone-flow-step-layout/standalone-flow-step-layout.svelte';
  import AccountBox from '$lib/components/account-box/account-box.svelte';
  import { createEventDispatcher, onMount } from 'svelte';
  import { makeTransactPayload, type StepComponentEvents } from '$lib/components/stepper/types';
  import PenIcon from '$lib/components/icons/Pen.svelte';
  import ListIcon from '$lib/components/icons/DripList.svelte';
  import TransactionsIcon from '$lib/components/icons/Transactions.svelte';
  import type { Writable } from 'svelte/store';
  import unreachable from '$lib/utils/unreachable';
  import formatTokenAmount from '$lib/utils/format-token-amount';
  import tokensStore from '$lib/stores/tokens/tokens.store';
  import CoinIcon from '$lib/components/icons/Coin.svelte';
  import WalletIcon from '$lib/components/icons/Wallet.svelte';
  import DripListService from '$lib/utils/driplist/DripListService';
  import type { State } from '../../create-drip-list-flow';
  import ListEditor from '$lib/components/list-editor/list-editor.svelte';
  import expect from '$lib/utils/expect';
  import Pause from '$lib/components/icons/Pause.svelte';
  import ContinuousSupportReviewCard from './components/continuous-support-review-card.svelte';
  import TokenStreams from '$lib/components/icons/TokenStreams.svelte';
  import { gql } from 'graphql-request';
  import query from '$lib/graphql/dripsQL';
  import type {
    DripListExistsQuery,
    DripListExistsQueryVariables,
  } from './__generated__/gql.generated';
  import OneTimeDonationReviewCard from './components/one-time-donation-review-card.svelte';
  import Heart from '$lib/components/icons/Heart.svelte';
  import network from '$lib/stores/wallet/network';
  import { invalidateAll } from '$lib/stores/fetched-data-cache/invalidate';
  import invalidateAccountCache from '$lib/utils/cache/remote/invalidate-account-cache';
  import WhatsNextCard from '$lib/components/whats-next/whats-next-card.svelte';
  import WhatsNextSection from '$lib/components/whats-next/whats-next-section.svelte';
  import WhatsNextItem from '$lib/components/whats-next/whats-next-item.svelte';

  const dispatch = createEventDispatcher<StepComponentEvents>();

  export let context: Writable<State>;
  export let connectedWalletHidden = false;

  let dripListService: DripListService;

  onMount(async () => {
    dripListService = await DripListService.new();
  });

  async function createDripList() {
    dispatch(
      'transact',
      makeTransactPayload({
        headline: 'Create your Drip List',
        messages: {
          duringBefore: 'Preparing Drip List creation transactions…',
        },
        before: async () => {
          return await dripListService.buildTransactContext({
            listTitle: $context.dripList.title,
            listDescription: $context.dripList.description,
            isVisible: true,
            weights: $context.dripList.weights,
            items: $context.dripList.items,
            support: (() => {
              if ($context.selectedSupportOption === 1) {
                return {
                  type: 'continuous',
                  topUpAmount:
                    $context.continuousSupportConfig.topUpAmountValueParsed ?? unreachable(),
                  amountPerSec:
                    ($context.continuousSupportConfig.streamRateValueParsed ?? unreachable()) /
                    BigInt(2592000), // 30 days in seconds
                  tokenAddress: $context.continuousSupportConfig.listSelected[0] ?? unreachable(),
                };
              } else if ($context.selectedSupportOption === 2) {
                return {
                  type: 'one-time',
                  donationAmount: $context.oneTimeDonationConfig.amount ?? unreachable(),
                  tokenAddress:
                    $context.oneTimeDonationConfig.selectedTokenAddress?.[0] ?? unreachable(),
                };
              }
            })(),
          });
        },

        transactions: ({ txs }) => txs,

        after: async (_, { dripListId }) => {
          const dripListExistsQuery = gql`
            query DripListExists($id: ID!, $chain: SupportedChain!) {
              dripList(id: $id, chain: $chain) {
                account {
                  accountId
                }
                isVisible
              }
            }
          `;

          const tryFetchList = async (listId: string) => {
            try {
              return await query<DripListExistsQuery, DripListExistsQueryVariables>(
                dripListExistsQuery,
                {
                  id: listId,
                  chain: network.gqlName,
                },
              );
            } catch {
              return false;
            }
          };

          await expect(
            () => tryFetchList(dripListId),
            (result) =>
              typeof result === 'boolean' ? result : Boolean(result.dripList?.isVisible),
            120000,
            1000,
          );

          await invalidateAccountCache(dripListId);
          await invalidateAll();

          $context.dripListId = dripListId;
        },
      }),
    );
  }

  function goBack() {
    dispatch('goBackward');
  }
</script>

<StandaloneFlowStepLayout headline="Review" description="Here's everything you set up.">
  <FormField type="div" title="Your Drip List">
    <svelte:fragment slot="action">
      <Button variant="ghost" on:click={() => dispatch('goForward', { by: -4 })} icon={PenIcon}
        >Edit</Button
      >
    </svelte:fragment>
    <h2 class="pixelated drip-list-title">{$context.dripList.title}</h2>
    {#if ($context.dripList.description ?? '').length > 0}
      <p class="my-4">{$context.dripList.description}</p>
    {/if}
    <ListEditor
      bind:weights={$context.dripList.weights}
      bind:items={$context.dripList.items}
      isEditable={false}
    />
  </FormField>

  {#if $context.selectedSupportOption !== undefined}
    <FormField type="div" title="Your support">
      <svelte:fragment slot="action">
        <Button variant="ghost" on:click={() => dispatch('goForward', { by: -3 })} icon={PenIcon}
          >Edit</Button
        >
      </svelte:fragment>
      {#if $context.selectedSupportOption === 1}
        <div class="card">
          <ContinuousSupportReviewCard
            topUpAmountValueParsed={$context.continuousSupportConfig.topUpAmountValueParsed ?? 0n}
            tokenAddress={$context.continuousSupportConfig.listSelected[0] ?? unreachable()}
            streamRateValueParsed={$context.continuousSupportConfig.streamRateValueParsed ?? 0n}
          />
        </div>
      {:else if $context.selectedSupportOption === 2}
        <div class="card">
          <OneTimeDonationReviewCard
            amount={$context.oneTimeDonationConfig.amount ?? unreachable()}
            tokenAddress={$context.oneTimeDonationConfig.selectedTokenAddress?.[0] ?? unreachable()}
          />
        </div>
      {/if}
    </FormField>
  {/if}

  {#if !connectedWalletHidden}
    <FormField type="div" title="Your connected wallet">
      <svelte:fragment slot="action">
        <Button variant="ghost" on:click={() => dispatch('goForward', { by: -4 })} icon={PenIcon}
          >Edit</Button
        >
      </svelte:fragment>
      <AccountBox hideDisconnect />
    </FormField>
  {/if}

  <WhatsNextSection>
    <WhatsNextCard>
      <svelte:fragment slot="title">On transaction confirmation…</svelte:fragment>
      <svelte:fragment slot="items">
        {#if $context.selectedSupportOption === 1}
          {#if $context.continuousSupportConfig.topUpAmountValueParsed && $context.continuousSupportConfig.streamRateValueParsed && $context.continuousSupportConfig.listSelected[0]}
            {@const token =
              tokensStore.getByAddress($context.continuousSupportConfig.listSelected[0]) ??
              unreachable()}
            <WhatsNextItem icon={TransactionsIcon}>
              <span class="typo-text-bold">
                {formatTokenAmount(
                  $context.continuousSupportConfig.topUpAmountValueParsed,
                  token.info.decimals,
                  1n,
                  false,
                )}
                {token.info.symbol} will be transferred from your wallet into your Drips account</span
              >
              and immediately begin streaming to your Drip List recipients at a rate of
              <span class="typo-text-bold">
                {formatTokenAmount(
                  $context.continuousSupportConfig.streamRateValueParsed,
                  token.info.decimals,
                  undefined,
                  false,
                )}
                {token.info.symbol} per month</span
              >.
            </WhatsNextItem>
          {/if}
        {/if}
        {#if $context.selectedSupportOption === 2}
          {@const token =
            tokensStore.getByAddress(
              $context.oneTimeDonationConfig.selectedTokenAddress?.[0] ?? unreachable(),
            ) ?? unreachable()}
          <WhatsNextItem icon={TransactionsIcon}>
            <span class="typo-text-bold">
              {formatTokenAmount(
                $context.oneTimeDonationConfig.amount ?? unreachable(),
                token.info.decimals,
                1n,
                false,
              )}
              {token.info.symbol} will be transferred from your wallet</span
            >
            and distributed to your list.
          </WhatsNextItem>
        {/if}
        <WhatsNextItem icon={ListIcon}
          >Your new Drip List will appear on your <span class="typo-text-bold">public profile</span
          >.</WhatsNextItem
        >
      </svelte:fragment>
    </WhatsNextCard>
    <WhatsNextCard>
      <svelte:fragment slot="title">After transaction confirmation…</svelte:fragment>
      <svelte:fragment slot="items">
        {#if $context.selectedSupportOption === 1}
          <WhatsNextItem icon={CoinIcon}>
            <span class="typo-text-bold">Add funds</span> (or withdraw any unstreamed funds) from your
            Drips dashboard.
          </WhatsNextItem>
          <WhatsNextItem icon={Pause}>
            <span class="typo-text-bold">Pause or cancel</span> your support anytime.
          </WhatsNextItem>
          <WhatsNextItem icon={Heart}>
            <span class="typo-text-bold">Create additional support streams</span> or make one-time donations
            anytime.
          </WhatsNextItem>
        {/if}
        {#if $context.selectedSupportOption === 2}
          <WhatsNextItem icon={Heart}>
            <span class="typo-text-bold">Make more one-time donations</span> or start a continuous support
            stream anytime.
          </WhatsNextItem>
        {/if}
        {#if $context.selectedSupportOption === undefined}
          <WhatsNextItem icon={TokenStreams}>
            You can <span class="typo-text-bold">start supporting</span> your Drip List anytime.
          </WhatsNextItem>
        {/if}
        <WhatsNextItem icon={PenIcon}
          ><span class="typo-text-bold">Edit</span> your Drip List anytime.</WhatsNextItem
        >
      </svelte:fragment>
    </WhatsNextCard>
  </WhatsNextSection>

  <svelte:fragment slot="left-actions">
    <Button icon={ArrowLeft} on:click={goBack}>Back</Button>
  </svelte:fragment>

  <svelte:fragment slot="actions">
    <Button icon={WalletIcon} variant="primary" on:click={createDripList}>Confirm in wallet</Button>
  </svelte:fragment>
</StandaloneFlowStepLayout>

<style>
  .drip-list-title {
    margin-bottom: 1rem;
  }
</style>
