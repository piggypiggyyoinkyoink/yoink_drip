<script lang="ts" context="module">
  import { gql } from 'graphql-request';

  export const DRIP_LIST_CARD_FRAGMENT = gql`
    ${EDIT_DRIP_LIST_FLOW_DRIP_LIST_FRAGMENT}
    ${SPLITS_COMPONENT_ADDRESS_RECEIVER_FRAGMENT}
    ${SPLITS_COMPONENT_PROJECT_RECEIVER_FRAGMENT}
    ${SPLITS_COMPONENT_DRIP_LIST_RECEIVER_FRAGMENT}
    ${PROJECT_AVATAR_FRAGMENT}
    ${SUPPORTER_PILE_FRAGMENT}
    ${CURRENT_AMOUNTS_TIMELINE_ITEM_FRAGMENT}
    fragment DripListCard on DripList {
      chain
      ...EditDripListFlowDripList
      name
      isVisible
      account {
        accountId
      }
      owner {
        accountId
        address
      }
      splits {
        ... on AddressReceiver {
          ...SplitsComponentAddressReceiver
        }
        ... on ProjectReceiver {
          ...SplitsComponentProjectReceiver
        }
        ... on DripListReceiver {
          ...SplitsComponentDripListReceiver
        }
      }
      totalEarned {
        tokenAddress
        amount
      }
      support {
        ...SupporterPile
        ... on StreamSupport {
          stream {
            timeline {
              ...CurrentAmountsTimelineItem
            }
            config {
              amountPerSecond {
                tokenAddress
              }
            }
          }
        }
      }
    }
  `;
</script>

<script lang="ts">
  import '$lib/utils/multiplayer/multiplayer';
  import Pen from '$lib/components/icons/Pen.svelte';
  import Button from '../button/button.svelte';
  import Drip from '../illustrations/drip.svelte';
  import Splits from '../splits/splits.svelte';
  import checkIsUser from '$lib/utils/check-is-user';
  import walletStore from '$lib/stores/wallet/wallet.store';
  import modal from '$lib/stores/modal';
  import Stepper from '../stepper/stepper.svelte';
  import editDripListSteps, {
    EDIT_DRIP_LIST_FLOW_DRIP_LIST_FRAGMENT,
  } from '$lib/flows/edit-drip-list/edit-members/edit-drip-list-steps';
  import ShareButton from '../share-button/share-button.svelte';
  import AggregateFiatEstimate from '../aggregate-fiat-estimate/aggregate-fiat-estimate.svelte';
  import { PROJECT_AVATAR_FRAGMENT } from '../project-avatar/project-avatar.svelte';
  import Pile from '../pile/pile.svelte';
  import { browser } from '$app/environment';
  import TextExpandable from '../text-expandable.svelte/text-expandable.svelte';
  import type { DripListCardFragment } from './__generated__/gql.generated';
  import getSupportersPile, { SUPPORTER_PILE_FRAGMENT } from './methods/get-supporters-pile';
  import IdentityBadge from '../identity-badge/identity-badge.svelte';
  import TabbedBox from '../tabbed-box/tabbed-box.svelte';
  import TransitionedHeight from '../transitioned-height/transitioned-height.svelte';
  import { fade } from 'svelte/transition';
  import type { VotingRound } from '$lib/utils/multiplayer/schemas';
  import unreachable from '$lib/utils/unreachable';
  import assert from '$lib/utils/assert';
  import VotingRoundSplits from './components/voting-round-splits.svelte';
  import Trash from '../icons/Trash.svelte';
  import deleteVotingRoundSteps from '$lib/flows/delete-voting-round/delete-voting-round-steps';
  import VotingRoundCollaborators from './components/voting-round-collaborators.svelte';
  import VotingRoundNoticeCard from './components/voting-round-notice-card.svelte';
  import VotingRoundCountdown from './components/voting-round-countdown.svelte';
  import Wallet from '../icons/Wallet.svelte';
  import publishVotingRoundListFlowSteps from '$lib/flows/publish-voting-round-list/publish-voting-round-list-flow-steps';
  import { BASE_URL } from '$lib/utils/base-url';
  import twemoji from '$lib/utils/twemoji';
  import {
    CURRENT_AMOUNTS_TIMELINE_ITEM_FRAGMENT,
    currentAmounts,
  } from '$lib/utils/current-amounts';
  import mapFilterUndefined from '$lib/utils/map-filter-undefined';
  import mergeAmounts from '$lib/utils/amounts/merge-amounts';
  import { onMount } from 'svelte';
  import tickStore from '$lib/stores/tick/tick.store';
  import StatusBadge from '../status-badge/status-badge.svelte';
  import Proposals from '../icons/Proposals.svelte';
  import PaddedHorizontalScroll from '../padded-horizontal-scroll/padded-horizontal-scroll.svelte';
  import ListEditor from '../list-editor/list-editor.svelte';
  import type { Items } from '../list-editor/types';
  import FormField from '../form-field/form-field.svelte';
  import contractConstants from '$lib/utils/sdk/utils/contract-constants';
  import {
    SPLITS_COMPONENT_ADDRESS_RECEIVER_FRAGMENT,
    SPLITS_COMPONENT_PROJECT_RECEIVER_FRAGMENT,
    SPLITS_COMPONENT_DRIP_LIST_RECEIVER_FRAGMENT,
    type SplitsComponentSplitsReceiver,
  } from '../splits/types';
  import { invalidateAll } from '$lib/stores/fetched-data-cache/invalidate';
  import type { SupportedChain } from '$lib/graphql/__generated__/base-types';
  import { NETWORK_CONFIG } from '$lib/stores/wallet/network';

  export let data: {
    dripList?: DripListCardFragment | null;
    votingRound?:
      | (VotingRound & {
          splits?: SplitsComponentSplitsReceiver[];
          allowedReceiversListEditorItems?: Items;
        })
      | null;
  };
  $: {
    assert(
      data.dripList || data.votingRound,
      'DripListCard requires either a dripList or a votingRound, or both',
    );
  }

  /** Minimal version w/ link to Drip List page for listing contexts */
  export let listingMode = false;

  export let isHidden = false;
  export let hideTotal = false;
  export let hideSupporterPile = false;
  export let hideDescription = false;
  export let clampTitle = true;
  export let openInNewTab = false;
  export let maxSplitRows: number | undefined = undefined;
  export let chainOverride: SupportedChain | undefined = undefined;

  $: dripList = data.dripList;
  $: votingRound = data.votingRound;

  $: listOwner = dripList?.owner;
  $: isOwnList = dripList && $walletStore && checkIsUser(dripList.owner.accountId);

  $: title = (dripList?.name || votingRound?.name) ?? unreachable();
  $: description = (dripList?.description || votingRound?.description) ?? '';

  $: supportersPile = dripList && getSupportersPile(dripList.support, 'tiny');

  function triggerEditModal() {
    if (!dripList) return;
    modal.show(Stepper, undefined, editDripListSteps(dripList));
  }

  let activeTab: string;
  $: {
    if (dripList && votingRound) {
      activeTab = 'tab-1';
    } else if (dripList) {
      activeTab = 'tab-1';
    } else if (votingRound) {
      activeTab = 'tab-2';
    }
  }

  $: isOwnVotingRound = votingRound?.publisherAddress === $walletStore?.address;

  let incomingStreamsTotalStreamed: { tokenAddress: string; amount: bigint }[];
  function updateIncomingStreamsTotalStreamed() {
    if (!dripList) return;

    const incomingStreams = mapFilterUndefined(dripList.support, (s) => {
      if (s.__typename === 'StreamSupport') {
        return s.stream;
      }
    });

    incomingStreamsTotalStreamed = mergeAmounts(
      incomingStreams.map((stream) => {
        const amount = currentAmounts(
          stream.timeline,
          stream.config.amountPerSecond.tokenAddress,
        ).currentAmount;

        return {
          tokenAddress: amount.tokenAddress,
          amount: amount.amount / BigInt(contractConstants.AMT_PER_SEC_MULTIPLIER),
        };
      }),
    );
  }
  onMount(() => {
    const tick = tickStore.register(updateIncomingStreamsTotalStreamed);
    return () => tickStore.deregister(tick);
  });

  $: totalEarned = mergeAmounts(incomingStreamsTotalStreamed ?? [], dripList?.totalEarned ?? []);

  $: urlBase = chainOverride
    ? `https://${Object.values(NETWORK_CONFIG).find((n) => n.gqlName === chainOverride)?.subdomain}`
    : '';

  $: dripListUrl = dripList
    ? `${urlBase}/app/drip-lists/${dripList.account.accountId}`
    : votingRound
      ? `${urlBase}/app/drip-lists/${votingRound.id}`
      : undefined;

  $: downloadableImageUrl = dripList
    ? `${urlBase}/api/share-images/drip-list/${dripList.account.accountId}.png?target=og`
    : votingRound
      ? `${urlBase}/api/share-images/drip-list/${votingRound.id}.png?target=og`
      : undefined;

  $: votingEnded = votingRound
    ? new Date() >= new Date(votingRound.schedule.voting.endsAt)
    : undefined;
</script>

{#if !listingMode && votingRound}
  <VotingRoundNoticeCard {votingRound} />
{/if}

<svelte:element
  this={listingMode ? 'a' : 'section'}
  href={dripListUrl}
  target={openInNewTab ? '_blank' : undefined}
  class="drip-list-card rounded-drip-lg overflow-hidden shadow-low group"
  class:hidden-list={isHidden}
>
  <div class="flex flex-col gap-4">
    <header class="px-6 pt-6 flex flex-col gap-2 lg:gap-4">
      <div class="title-and-actions">
        <h1
          class:line-clamp-1={clampTitle}
          class=" title rounded twemoji-text"
          style:font-size={listingMode ? '28px' : undefined}
        >
          {@html twemoji(title)}
        </h1>
        {#if listingMode && votingRound}
          <StatusBadge icon={Proposals} size="small" color={votingEnded ? 'foreground' : 'primary'}>
            {votingEnded ? 'Voting ended' : 'In voting'}
          </StatusBadge>
        {/if}
        {#if !listingMode}
          <div class="flex items-center gap-2 -my-1">
            <ShareButton
              buttonVariant="normal"
              url="{BASE_URL}{dripListUrl}"
              {downloadableImageUrl}
              disabled={!dripList?.isVisible}
            />
            {#if isOwnList}
              <Button disabled={!dripList?.isVisible} on:click={triggerEditModal} icon={Pen}
                >Edit list</Button
              >
            {/if}
          </div>
        {/if}
      </div>
      {#if !hideDescription && description.length > 0}
        <div class="description twemoji-text">
          <TextExpandable numberOfLines={listingMode ? 2 : 4} isExpandable={!listingMode}>
            {@html twemoji(description)}
          </TextExpandable>
        </div>
      {/if}
      {#if !listingMode}
        <div class="flex gap-2">
          Created by <IdentityBadge
            showAvatar={true}
            showIdentity={true}
            address={listOwner?.address ?? votingRound?.publisherAddress ?? unreachable()}
            disableTooltip={true}
          />
        </div>
      {/if}
    </header>

    <section>
      {#if !listingMode && dripList && votingRound}
        <div class="-mt-4 mb-10 sm:-mt-6 sm:mb-8">
          <TabbedBox
            bind:activeTab
            tabs={{
              1: 'Current',
              2: 'In voting',
            }}
            ariaLabel="Toggle list versions"
          />
        </div>
      {/if}

      <TransitionedHeight transitionHeightChanges>
        <div class="tabs">
          <div class="list tab tab-1" class:active-tab={activeTab === 'tab-1'}>
            {#if dripList}
              <div class="flex flex-col gap-1.5">
                <div class="totals">
                  <div class="drip-icon">
                    <Drip />
                  </div>
                  {#if !hideTotal}
                    <div class="typo-text tabular-nums total-streamed-badge">
                      {#if browser}
                        <AggregateFiatEstimate supressUnknownAmountsWarning amounts={totalEarned} />
                      {/if}
                      <span class="muted">&nbsp;total</span>
                    </div>
                  {/if}
                  {#if !hideSupporterPile && supportersPile && supportersPile.length > 0}
                    <div
                      in:fade={{ duration: 300 }}
                      class="hide-on-mobile flex items-center gap-1.5 min-w-0"
                    >
                      <span class="typo-text-small truncate muted">Supported by</span>
                      <Pile
                        maxItems={3}
                        components={supportersPile ?? []}
                        itemsClickable={!listingMode}
                      />
                    </div>
                  {/if}
                </div>
                <PaddedHorizontalScroll disableScroll={listingMode}>
                  <div
                    class="splits-component"
                    style:pointer-events={listingMode ? 'none' : undefined}
                  >
                    <Splits
                      disableLinks={listingMode}
                      maxRows={maxSplitRows ??
                        (listingMode
                          ? dripList.description && !hideDescription
                            ? 3
                            : 4
                          : undefined)}
                      groupsExpandable={!listingMode}
                      list={dripList.splits}
                      {chainOverride}
                    />
                  </div>
                </PaddedHorizontalScroll>
              </div>
            {/if}
          </div>

          <div
            class="list tab tab-2"
            class:active-tab={activeTab === 'tab-2'}
            class:-mt-6={!votingRound?.result}
          >
            {#if votingRound}
              <div class="flex flex-col gap-1.5">
                <div class="splits">
                  <div class="splits-component">
                    <VotingRoundSplits
                      {listingMode}
                      maxRows={listingMode ? (votingRound.description ? 3 : 4) : undefined}
                      {votingRound}
                    />
                  </div>
                </div>
              </div>

              {#if !listingMode}
                {#if votingRound.allowedReceivers?.length && votingRound.status === 'Started'}
                  <FormField title="Possible recipients" type="div">
                    <ListEditor
                      items={votingRound.allowedReceiversListEditorItems}
                      weightsMode={false}
                      isEditable={false}
                    />
                  </FormField>
                {/if}

                <VotingRoundCollaborators {votingRound} />

                <VotingRoundCountdown {votingRound} on:end={invalidateAll} />

                {#if isOwnVotingRound}
                  <div class="actions">
                    <div>
                      <Button
                        icon={Trash}
                        on:click={() =>
                          modal.show(
                            Stepper,
                            undefined,
                            deleteVotingRoundSteps(votingRound?.id ?? unreachable()),
                          )}>Delete voting round</Button
                      >
                    </div>

                    {#if votingRound.status === 'Completed' || votingRound.status === 'PendingLinkCompletion'}
                      <div>
                        <Button
                          variant="primary"
                          icon={Wallet}
                          disabled={!votingRound.result}
                          on:click={() =>
                            modal.show(
                              Stepper,
                              undefined,
                              votingRound
                                ? publishVotingRoundListFlowSteps(
                                    votingRound.id,
                                    votingRound.name,
                                    votingRound.description ?? undefined,
                                  )
                                : unreachable(),
                            )}>Publish Drip List</Button
                        >
                      </div>
                    {/if}
                  </div>
                {/if}
              {/if}
            {/if}
          </div>
        </div>
      </TransitionedHeight>
    </section>
  </div>
</svelte:element>

<style>
  .drip-list-card {
    box-shadow: var(--elevation-low);
    transition:
      transform 0.2s,
      box-shadow 0.2s;
    position: relative;
    background: var(--color-background);
  }

  a.drip-list-card:hover,
  a.drip-list-card:focus-visible {
    box-shadow: var(--elevation-medium);
    transform: translateY(-0.125rem);
  }

  .drip-list-card .title-and-actions {
    display: flex;
    justify-content: space-between;
    gap: 1rem;
    text-align: left;
  }

  .drip-list-card .title-and-actions {
    flex-wrap: wrap;
  }

  .drip-list-card .title-and-actions h1 {
    min-width: fit-content;
  }

  .totals {
    display: flex;
    align-items: center;
    gap: 1rem;
    position: relative;
  }

  .totals .drip-icon {
    width: 1.5rem;
  }

  .totals .total-streamed-badge {
    background-color: var(--color-foreground-level-2);
    border-radius: 1rem 0 1rem 1rem;
    display: flex;
    align-items: center;
    padding: 0.25rem 0.5rem;
  }

  .list {
    padding: 0 1.5rem 1.5rem 1.5rem;
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
  }

  .splits {
    overflow-x: scroll;
  }

  .splits-component {
    margin-left: 10px;
  }

  .muted {
    color: var(--color-foreground-level-5);
  }

  .tabs {
    position: relative;
  }

  .tab {
    position: absolute;
    top: 0;
    transition:
      opacity 0.3s,
      transform 0.3s;
    opacity: 0;
    pointer-events: none;
  }

  .tab-1 {
    transform: translateX(-4rem);
  }

  .tab-2 {
    transform: translateX(4rem);
  }

  .tab.active-tab {
    position: relative;
    transform: none;
    opacity: 1;
    pointer-events: auto;
  }

  .actions {
    display: flex;
    gap: 0.5rem;
    justify-content: flex-end;
    margin-top: 1rem;
  }

  @media (max-width: 768px) {
    .actions {
      flex-direction: column;
    }

    .hide-on-mobile {
      display: none;
    }
  }

  .hidden-list {
    color: var(--color-foreground);
    opacity: 0;
    animation: fadeIn 1s ease forwards;
  }

  @keyframes fadeIn {
    to {
      opacity: 0.3;
    }
  }
</style>
