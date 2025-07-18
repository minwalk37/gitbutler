<script lang="ts">
	import { goto } from '$app/navigation';
	import ReduxResult from '$components/ReduxResult.svelte';
	import Resizer from '$components/Resizer.svelte';
	import Scrollbar from '$components/Scrollbar.svelte';
	import BranchExplorer, { type SelectedOption } from '$components/v3/BranchExplorer.svelte';
	import BranchView from '$components/v3/BranchView.svelte';
	import BranchesViewBranch from '$components/v3/BranchesViewBranch.svelte';
	import BranchesViewPr from '$components/v3/BranchesViewPR.svelte';
	import BranchesViewStack from '$components/v3/BranchesViewStack.svelte';
	import MainViewport from '$components/v3/MainViewport.svelte';
	import PrBranchView from '$components/v3/PRBranchView.svelte';
	import SelectionView from '$components/v3/SelectionView.svelte';
	import TargetCommitList from '$components/v3/TargetCommitList.svelte';
	import UnappliedBranchView from '$components/v3/UnappliedBranchView.svelte';
	import UnappliedCommitView from '$components/v3/UnappliedCommitView.svelte';
	import BranchListCard from '$components/v3/branchesPage/BranchListCard.svelte';
	import BranchesListGroup from '$components/v3/branchesPage/BranchesListGroup.svelte';
	import CurrentOriginCard from '$components/v3/branchesPage/CurrentOriginCard.svelte';
	import PRListCard from '$components/v3/branchesPage/PRListCard.svelte';
	import BaseBranchService from '$lib/baseBranch/baseBranchService.svelte';
	import { isParsedError } from '$lib/error/parser';
	import { workspacePath } from '$lib/routes/routes.svelte';
	import { StackService } from '$lib/stacks/stackService.svelte';
	import { UiState } from '$lib/state/uiState.svelte';
	import { TestId } from '$lib/testing/testIds';
	import { inject } from '@gitbutler/shared/context';
	import { persisted } from '@gitbutler/shared/persisted';
	import AsyncButton from '@gitbutler/ui/AsyncButton.svelte';
	import Button from '@gitbutler/ui/Button.svelte';
	import Modal from '@gitbutler/ui/Modal.svelte';
	import { getTimeAgo } from '@gitbutler/ui/utils/timeAgo';
	import type { SidebarEntrySubject } from '$lib/branches/branchListing';
	import type { SelectionId } from '$lib/selection/key';

	type Props = {
		projectId: string;
	};

	const { projectId }: Props = $props();

	const [uiState, stackService, baseBranchService] = inject(
		UiState,
		StackService,
		BaseBranchService
	);

	const projectState = $derived(uiState.project(projectId));
	const branchesState = $derived(projectState.branchesSelection);

	const baseBranchResult = $derived(baseBranchService.baseBranch(projectId));
	const branchesSelection = $derived(projectState.branchesSelection);
	const selectedOption = persisted<SelectedOption>('all', `branches-selectedOption-${projectId}`);

	let branchColumn = $state<HTMLDivElement>();
	let commitColumn = $state<HTMLDivElement>();
	let previewColumn = $state<HTMLDivElement>();
	let rightWrapper = $state<HTMLDivElement>();

	const selectionId: SelectionId | undefined = $derived.by(() => {
		const current = branchesState?.current;
		if (current.commitId) {
			return { type: 'commit', commitId: current.commitId };
		}
		if (current.branchName) {
			const branchName = current.remote
				? current.remote + '/' + current.branchName
				: current.branchName;
			if (current.stackId) {
				return {
					type: 'branch',
					branchName,
					stackId: current.stackId
				};
			}
			return {
				type: 'branch',
				branchName
			};
		}
	});

	async function checkoutBranch() {
		const { branchName, remote, prNumber, hasLocal } = branchesState.current;
		const remoteRef = remote ? `refs/remotes/${remote}/${branchName}` : undefined;
		const branchRef = hasLocal ? `refs/heads/${branchName}` : remoteRef;
		if (branchRef) {
			await stackService.createVirtualBranchFromBranch({
				projectId,
				branch: branchRef,
				remote: remoteRef,
				prNumber
			});
			await baseBranchService.refreshBaseBranch(projectId);
		}
		goto(workspacePath(projectId));
	}

	async function deleteLocalBranch(branchName: string) {
		const hasLocal = branchesState.current.hasLocal;
		if (!hasLocal) {
			return;
		}

		await stackService.deleteLocalBranch({
			projectId,
			refname: `refs/heads/${branchName}`,
			givenName: branchName
		});

		// Unselect branch
		branchesSelection.set({});
		await baseBranchService.refreshBaseBranch(projectId);
	}

	let prBranch = $state<BranchesViewPr>();

	function applyFromFork() {
		prBranch?.applyPr();
	}

	let deleteLocalBranchModal = $state<Modal>();

	function handleDeleteLocalBranch(branchName: string) {
		deleteLocalBranchModal?.show(branchName);
	}

	function onerror(err: unknown) {
		// Clear selection if branch not found.
		if (isParsedError(err) && err.code === 'errors.branch.notfound') {
			branchesSelection.set({});
			console.warn('Branches selection cleared');
		}
	}
</script>

<Modal
	testId={TestId.DeleteLocalBranchConfirmationModal}
	bind:this={deleteLocalBranchModal}
	title="Delete local branch"
	width="small"
	defaultItem={branchesState.current.branchName}
	onSubmit={async (close, branchName: string | undefined) => {
		if (branchName) {
			await deleteLocalBranch(branchName);
		}
		close();
	}}
>
	{#snippet children(branchName)}
		<p>Are you sure you want to delete the local changes inside the branch {branchName}?</p>
	{/snippet}

	{#snippet controls(close)}
		<Button
			testId={TestId.DeleteLocalBranchConfirmationModal_Cancel}
			kind="outline"
			type="reset"
			onclick={close}>Cancel</Button
		>
		<Button
			testId={TestId.DeleteLocalBranchConfirmationModal_Delete}
			style="error"
			type="submit"
			icon="bin">Delete</Button
		>
	{/snippet}
</Modal>

<ReduxResult {projectId} result={baseBranchResult.current}>
	{#snippet children(baseBranch)}
		{@const lastCommit = baseBranch.recentCommits.at(0)}
		{@const current = branchesState.current}
		{@const someBranchSelected = current.branchName !== undefined}
		{@const isTargetBranch = current.branchName === baseBranch.shortName}
		{@const inWorkspaceOrTargetBranch = current.inWorkspace || isTargetBranch}
		{@const isStackOrNormalBranchPreview =
			current.stackId || (current.branchName && isTargetBranch)}
		{@const isNonLocalPr = !isStackOrNormalBranchPreview && current.prNumber !== undefined}

		<MainViewport name="branches" leftResizerRadius leftWidth={{ default: 360, min: 280 }}>
			{#snippet left()}
				<BranchesListGroup title="Current workspace target">
					<!-- TODO: We need an API for `commitsCount`! -->
					<CurrentOriginCard
						originName={baseBranch.branchName}
						lastCommit={lastCommit
							? {
									author: lastCommit.author,
									ago: getTimeAgo(lastCommit.createdAt, true),
									branch: baseBranch.shortName,
									sha: lastCommit.id.slice(0, 7)
								}
							: undefined}
						onclick={() => {
							branchesSelection.set({ branchName: baseBranch.shortName, isTarget: true });
						}}
						selected={(branchesSelection.current.branchName === undefined ||
							branchesSelection.current.branchName === baseBranch.shortName) &&
							branchesSelection.current.prNumber === undefined}
					/>
				</BranchesListGroup>
				<BranchExplorer {projectId} bind:selectedOption={$selectedOption}>
					{#snippet sidebarEntry(sidebarEntrySubject: SidebarEntrySubject)}
						{#if sidebarEntrySubject.type === 'branchListing'}
							<BranchListCard
								{projectId}
								branchListing={sidebarEntrySubject.subject}
								prs={sidebarEntrySubject.prs}
								selected={sidebarEntrySubject.subject.stack
									? branchesSelection.current.branchName ===
										sidebarEntrySubject.subject.stack.branches.at(0)
									: branchesSelection.current.branchName === sidebarEntrySubject.subject.name}
								onclick={({ listing, pr }) => {
									if (listing.stack) {
										branchesSelection.set({
											stackId: listing.stack.id,
											branchName: listing.stack.branches.at(0),
											prNumber: pr?.number,
											inWorkspace: listing.stack.inWorkspace,
											hasLocal: listing.hasLocal
										});
									} else {
										branchesSelection.set({
											branchName: listing.name,
											prNumber: pr?.number,
											remote: listing.remotes.at(0),
											hasLocal: listing.hasLocal
										});
									}
								}}
							/>
						{:else}
							<PRListCard
								number={sidebarEntrySubject.subject.number}
								isDraft={sidebarEntrySubject.subject.draft}
								title={sidebarEntrySubject.subject.title}
								sourceBranch={sidebarEntrySubject.subject.sourceBranch}
								author={{
									name: sidebarEntrySubject.subject.author?.name,
									email: sidebarEntrySubject.subject.author?.email,
									gravatarUrl: sidebarEntrySubject.subject.author?.gravatarUrl
								}}
								modifiedAt={sidebarEntrySubject.subject.modifiedAt}
								selected={branchesSelection.current.prNumber === sidebarEntrySubject.subject.number}
								onclick={(pr) => branchesSelection.set({ prNumber: pr.number })}
								noRemote
							/>
						{/if}
					{/snippet}
				</BranchExplorer>
			{/snippet}

			{#snippet right()}
				<div class="right-wrapper hide-native-scrollbar" bind:this={rightWrapper}>
					<div class="branch-column" bind:this={branchColumn}>
						<!-- Apply branch -->
						{#if !inWorkspaceOrTargetBranch && someBranchSelected}
							{@const doesNotHaveLocalTooltip = current.hasLocal
								? undefined
								: 'No local branch to delete'}
							{@const doesNotHaveABranchNameTooltip = current.branchName
								? undefined
								: 'No branch selected to delete'}

							<div class="branches-actions">
								{#if !current.isTarget}
									<AsyncButton
										testId={TestId.BranchesViewApplyBranchButton}
										icon="workbench"
										action={async () => {
											await checkoutBranch();
										}}
									>
										Apply to workspace
									</AsyncButton>
								{/if}

								<Button
									testId={TestId.BranchesViewDeleteLocalBranchButton}
									kind="outline"
									icon="bin-small"
									onclick={() => {
										if (current.branchName) {
											handleDeleteLocalBranch(current.branchName);
										}
									}}
									disabled={!current.hasLocal || !current.branchName}
									tooltip={doesNotHaveLocalTooltip ?? doesNotHaveABranchNameTooltip}
								>
									Delete local
								</Button>
							</div>
						{/if}

						<!-- Apply PR (from fork) -->
						{#if isNonLocalPr && !inWorkspaceOrTargetBranch}
							<div class="branches-actions">
								{#if !current.isTarget}
									<Button
										testId={TestId.BranchesViewApplyFromForkButton}
										icon="workbench"
										onclick={applyFromFork}
									>
										Apply PR to workspace
									</Button>
								{/if}
							</div>
						{/if}

						<div class="commits dotted-container" class:target-branch={isTargetBranch}>
							{#if isTargetBranch || (current.branchName === undefined && current.prNumber === undefined)}
								<TargetCommitList {projectId} />
							{:else if current.stackId}
								<BranchesViewStack {projectId} stackId={current.stackId} {onerror} />
							{:else if current.branchName}
								<BranchesViewBranch
									{projectId}
									branchName={current.branchName}
									remote={current.remote}
									{onerror}
								/>
							{:else if current.prNumber}
								<BranchesViewPr
									bind:this={prBranch}
									{projectId}
									prNumber={current.prNumber}
									{onerror}
								/>
							{/if}
						</div>

						<Resizer
							viewport={branchColumn}
							persistId="branches-branch-column"
							direction="right"
							defaultValue={15}
							minWidth={10}
							maxWidth={30}
						/>
					</div>
					<div class="commit-column" bind:this={commitColumn}>
						{#if current.commitId}
							<UnappliedCommitView {projectId} commitId={current.commitId} />
						{:else if current.branchName}
							{#if current.inWorkspace && current.stackId}
								<BranchView
									{projectId}
									branchName={current.branchName}
									stackId={current.stackId}
									draggableFiles={false}
									active
									{onerror}
								/>
							{:else if !current.isTarget}
								<UnappliedBranchView
									{projectId}
									branchName={current.branchName}
									stackId={current.stackId}
									remote={current.remote}
									prNumber={current.prNumber}
									{onerror}
								/>
							{/if}
						{:else if current.prNumber}
							<PrBranchView {projectId} prNumber={current.prNumber} {onerror} />
						{/if}
						<Resizer
							viewport={commitColumn}
							persistId="branches-branch-column"
							direction="right"
							defaultValue={15}
							minWidth={10}
							maxWidth={30}
						/>
					</div>
					<div class="preview-column" bind:this={previewColumn}>
						<SelectionView {projectId} {selectionId} draggableFiles />
						<Resizer
							viewport={previewColumn}
							persistId="branches-preview-column"
							direction="right"
							defaultValue={35}
							minWidth={20}
							maxWidth={90}
						/>
					</div>
				</div>
				<Scrollbar viewport={rightWrapper} horz />
			{/snippet}
		</MainViewport>
	{/snippet}
</ReduxResult>

<style lang="postcss">
	.branch-column {
		display: flex;
		position: relative;
		flex-grow: 0;
		flex-shrink: 0;
		flex-direction: column;
		max-height: calc(100% + 1px);
		border-right: 1px solid var(--clr-border-2);
	}

	.commit-column {
		position: relative;
		flex-grow: 0;
		flex-shrink: 0;
		max-height: calc(100% + 1px);
		overflow: hidden;
		border-right: 1px solid var(--clr-border-2);
	}

	.preview-column {
		position: relative;
		flex-grow: 0;
		flex-shrink: 0;
		max-height: calc(100% + 1px);
		overflow: hidden;
	}

	.commits {
		display: flex;
		position: relative;
		flex: 1;
		flex-direction: column;
		overflow: hidden;
	}

	.branches-actions {
		display: flex;
		padding: 12px;
		gap: 6px;
		border-bottom: 1px solid var(--clr-border-2);
		background-color: var(--clr-bg-1);
	}

	/* MODIFIERS */
	.dotted-container {
		padding: 12px;
		border-radius: 0 0 var(--radius-ml) var(--radius-ml);
		&.target-branch {
			padding: 0;
			border-radius: 0;
		}
	}

	.right-wrapper {
		display: flex;
		position: relative;
		height: 100%;
		overflow: hidden;
		overflow-x: auto;
	}
</style>
