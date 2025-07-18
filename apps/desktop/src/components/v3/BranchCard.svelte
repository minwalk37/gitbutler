<script lang="ts">
	import CardOverlay from '$components/CardOverlay.svelte';
	import Dropzone from '$components/Dropzone.svelte';
	import BranchBadge from '$components/v3/BranchBadge.svelte';
	import BranchDividerLine from '$components/v3/BranchDividerLine.svelte';
	import BranchHeader from '$components/v3/BranchHeader.svelte';
	import BranchHeaderContextMenu from '$components/v3/BranchHeaderContextMenu.svelte';
	import PrNumberUpdater from '$components/v3/PrNumberUpdater.svelte';
	import ReviewView from '$components/v3/ReviewView.svelte';
	import { MoveCommitDzHandler } from '$lib/commits/dropHandler';
	import { StackService } from '$lib/stacks/stackService.svelte';
	import { UiState } from '$lib/state/uiState.svelte';
	import { TestId } from '$lib/testing/testIds';
	import { inject } from '@gitbutler/shared/context';
	import ReviewBadge from '@gitbutler/ui/ReviewBadge.svelte';
	import { getTimeAgo } from '@gitbutler/ui/utils/timeAgo';
	import type { PushStatus } from '$lib/stacks/stack';
	import type iconsJson from '@gitbutler/ui/data/icons.json';
	import type { Snippet } from 'svelte';

	interface BranchCardProps {
		type: 'draft-branch' | 'normal-branch' | 'stack-branch' | 'pr-branch';
		projectId: string;
		branchName: string;
		isCommitting?: boolean;
		expand?: boolean;
		lineColor: string;
		readonly: boolean;
		first?: boolean;
		active?: boolean;
	}

	interface DraftBranchProps extends BranchCardProps {
		type: 'draft-branch';
		branchContent: Snippet;
		buttons?: Snippet;
	}

	interface NormalBranchProps extends BranchCardProps {
		type: 'normal-branch';
		iconName: keyof typeof iconsJson;
		selected: boolean;
		trackingBranch?: string;
		lastUpdatedAt?: number;
		isTopBranch?: boolean;
		isNewBranch?: boolean;
		onclick: () => void;
		branchContent: Snippet;
	}

	interface StackBranchProps extends BranchCardProps {
		type: 'stack-branch';
		iconName: keyof typeof iconsJson;
		stackId: string;
		selected: boolean;
		trackingBranch?: string;
		isNewBranch?: boolean;
		prNumber?: number;
		reviewId?: string;
		pushStatus: PushStatus;
		lastUpdatedAt?: number;
		isConflicted: boolean;
		contextMenu?: typeof BranchHeaderContextMenu;
		onclick: () => void;
		menu?: Snippet<[{ rightClickTrigger: HTMLElement }]>;
		buttons?: Snippet;
		branchContent: Snippet;
	}

	interface PrBranchProps extends BranchCardProps {
		type: 'pr-branch';
		selected: boolean;
		trackingBranch: string;
		lastUpdatedAt: number;
	}

	type Props = DraftBranchProps | NormalBranchProps | StackBranchProps | PrBranchProps;

	let { projectId, branchName, expand, active, lineColor, readonly, ...args }: Props = $props();

	const [uiState, stackService] = inject(UiState, StackService);

	const [updateName, nameUpdate] = stackService.updateBranchName;

	const stackState = $derived(
		args.type === 'stack-branch' ? uiState.stack(args.stackId) : undefined
	);
	const selection = $derived(stackState ? stackState.selection.current : undefined);
	const selected = $derived(selection?.branchName === branchName);
	const isPushed = $derived(!!(args.type === 'draft-branch' ? undefined : args.trackingBranch));

	async function updateBranchName(title: string) {
		if (args.type === 'draft-branch') {
			uiState.global.draftBranchName.set(title);
			const normalized = await stackService.normalizeBranchName(title);
			if (normalized.data) {
				uiState.global.draftBranchName.set(normalized.data);
			}
		} else if (args.type === 'stack-branch') {
			updateName({
				projectId,
				stackId: args.stackId,
				branchName,
				newName: title
			});
		}
	}
</script>

{#if ((args.type === 'stack-branch' && !args.first) || (args.type === 'normal-branch' && !args.first)) && lineColor}
	<BranchDividerLine {lineColor} />
{/if}

<div
	class="branch-card"
	class:selected
	class:draft={args.type === 'draft-branch'}
	class:expand
	data-series-name={branchName}
	data-testid={TestId.BranchCard}
>
	{#if args.type === 'stack-branch'}
		{@const moveHandler = new MoveCommitDzHandler(stackService, args.stackId, projectId)}
		{#if !args.prNumber}
			<PrNumberUpdater {projectId} stackId={args.stackId} {branchName} />
		{/if}
		<Dropzone handlers={[moveHandler]}>
			{#snippet overlay({ hovered, activated, handler })}
				{@const label = handler instanceof MoveCommitDzHandler ? 'Move here' : 'Start commit'}
				<CardOverlay {hovered} {activated} {label} />
			{/snippet}

			<BranchHeader
				{branchName}
				isEmpty={args.isNewBranch}
				selected={args.selected}
				selectIndicator
				draft={false}
				{lineColor}
				isCommitting={args.isCommitting}
				iconName={args.iconName}
				{updateBranchName}
				isUpdatingName={nameUpdate.current.isLoading}
				{active}
				{readonly}
				{isPushed}
				onclick={args.onclick}
				menu={args.menu}
				buttons={args.buttons}
			>
				{#snippet emptyState()}
					<span class="branch-header__empty-state-span">This is an empty branch.</span>
					<span class="branch-header__empty-state-span">Click for details.</span>
					<br />
					Create or drag & drop commits here.
				{/snippet}
				{#snippet content()}
					<span class="branch-header__item">
						<BranchBadge pushStatus={args.pushStatus} unstyled />
					</span>
					<span class="branch-header__divider">•</span>

					{#if args.isConflicted}
						<span class="branch-header__item branch-header__item--conflict"> Conflicts </span>
						<span class="branch-header__divider">•</span>
					{/if}

					{#if args.lastUpdatedAt}
						<span class="branch-header__item">
							{getTimeAgo(new Date(args.lastUpdatedAt))}
						</span>
					{/if}

					{#if args.reviewId || args.prNumber}
						<span class="branch-header__divider">•</span>
						<div class="branch-header__review-badges">
							{#if args.reviewId}
								<ReviewBadge brId={args.reviewId} brStatus="unknown" />
							{/if}
							{#if args.prNumber}
								<ReviewBadge prNumber={args.prNumber} prStatus="unknown" />
							{/if}
						</div>
					{/if}
				{/snippet}
			</BranchHeader>
		</Dropzone>
		{#if stackState?.action.current === 'review'}
			<div class="review-wrapper">
				<ReviewView
					{projectId}
					{branchName}
					stackId={args.stackId}
					oncancel={() => stackState.action.set(undefined)}
				/>
			</div>
		{/if}
	{:else if args.type === 'normal-branch'}
		<BranchHeader
			{branchName}
			isEmpty={args.isNewBranch}
			selected={args.selected}
			selectIndicator
			draft={false}
			{lineColor}
			iconName={args.iconName}
			{updateBranchName}
			isUpdatingName={nameUpdate.current.isLoading}
			{active}
			readonly
			{isPushed}
			onclick={args.onclick}
		>
			{#snippet emptyState()}
				<span class="branch-header__empty-state-span">There are no commits yet on this branch.</span
				>
			{/snippet}
			{#snippet content()}
				{#if args.lastUpdatedAt}
					<span class="branch-header__item">
						{getTimeAgo(new Date(args.lastUpdatedAt))}
					</span>
				{/if}
			{/snippet}
		</BranchHeader>
	{:else if args.type === 'pr-branch'}
		<BranchHeader
			{branchName}
			isEmpty
			selected={args.selected}
			selectIndicator
			draft={false}
			{lineColor}
			iconName="branch-remote"
			{updateBranchName}
			isUpdatingName={nameUpdate.current.isLoading}
			{active}
			readonly
			isPushed
		>
			{#snippet content()}
				{#if args.lastUpdatedAt}
					<span class="branch-header__item">
						{getTimeAgo(new Date(args.lastUpdatedAt))}
					</span>
				{/if}
			{/snippet}
		</BranchHeader>
	{:else}
		<BranchHeader
			{branchName}
			isEmpty
			selected
			selectIndicator={false}
			draft
			{lineColor}
			iconName="branch-local"
			{updateBranchName}
			isUpdatingName={nameUpdate.current.isLoading}
			{active}
			readonly={false}
			isPushed={false}
		>
			{#snippet emptyState()}
				A new branch will be created for your commit.
				<br />
				Click the name to rename it now or later.
			{/snippet}
		</BranchHeader>
	{/if}

	{#if args.type === 'stack-branch' || args.type === 'normal-branch' || args.type === 'draft-branch'}
		{@render args.branchContent()}
	{/if}
</div>

<style lang="postcss">
	.branch-card {
		display: flex;
		position: relative;
		flex-direction: column;
		width: 100%;
		overflow: hidden;
		border: 1px solid var(--clr-border-2);
		border-radius: var(--radius-ml);
		background: var(--clr-bg-1);
	}
	.expand {
		height: 100%;
	}

	.branch-header__item {
		color: var(--clr-text-2);
		white-space: nowrap;
	}

	.branch-header__item--conflict {
		color: var(--clr-theme-err-element);
	}

	.branch-header__divider {
		color: var(--clr-text-3);
	}

	.branch-header__empty-state-span {
		text-wrap: nowrap;
	}

	.branch-header__review-badges {
		display: flex;
		gap: 4px;
	}

	.review-wrapper {
		padding: 12px;
		border-bottom: 1px solid var(--clr-border-2);
	}
</style>
