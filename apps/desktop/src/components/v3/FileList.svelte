<!-- This is a V3 replacement for `BranchFileList.svelte` -->
<script lang="ts">
	import LazyloadContainer from '$components/LazyloadContainer.svelte';
	import FileListItemWrapper from '$components/v3/FileListItemWrapper.svelte';
	import FileTreeNode from '$components/v3/FileTreeNode.svelte';
	import DiffInputContext from '$lib/ai/diffInputContext.svelte';
	import AIMacros from '$lib/ai/macros.svelte';
	import { PromptService } from '$lib/ai/promptService';
	import { AIService } from '$lib/ai/service';
	import { projectAiGenEnabled } from '$lib/config/config';
	import { conflictEntryHint } from '$lib/conflictEntryPresence';
	import { abbreviateFolders, changesToFileTree, sortLikeFileTree } from '$lib/files/filetreeV3';
	import {
		type TreeChange,
		type Modification,
		previousPathBytesFromTreeChange
	} from '$lib/hunks/change';
	import { DiffService } from '$lib/hunks/diffService.svelte';
	import { showToast } from '$lib/notifications/toasts';
	import { IdSelection } from '$lib/selection/idSelection.svelte';
	import { selectFilesInList, updateSelection } from '$lib/selection/idSelectionUtils';
	import { type SelectionId } from '$lib/selection/key';
	import StackMacros from '$lib/stacks/macros';
	import {
		StackService,
		type CreateCommitRequestWorktreeChanges
	} from '$lib/stacks/stackService.svelte';
	import { UiState } from '$lib/state/uiState.svelte';
	import { chunk } from '$lib/utils/array';
	import { WorktreeService } from '$lib/worktree/worktreeService.svelte';
	import { inject } from '@gitbutler/shared/context';
	import FileListItemV3 from '@gitbutler/ui/file/FileListItemV3.svelte';
	import type { DiffInputContextArgs } from '$lib/ai/diffInputContext.svelte';
	import type { ConflictEntriesObj } from '$lib/files/conflicts';

	type Props = {
		projectId: string;
		stackId?: string;
		changes: TreeChange[];
		listMode: 'list' | 'tree';
		showCheckboxes?: boolean;
		selectionId: SelectionId;
		active?: boolean;
		conflictEntries?: ConflictEntriesObj;
		draggableFiles: boolean;
		onselect?: () => void;
	};

	const {
		projectId,
		changes,
		listMode,
		selectionId,
		showCheckboxes,
		active,
		stackId,
		conflictEntries,
		draggableFiles,
		onselect
	}: Props = $props();

	const [
		stackService,
		uiState,
		idSelection,
		aiService,
		promptService,
		diffService,
		worktreeService
	] = inject(
		StackService,
		UiState,
		IdSelection,
		AIService,
		PromptService,
		DiffService,
		WorktreeService
	);

	let currentDisplayIndex = $state(0);

	const sortedChanges = $derived(sortLikeFileTree(changes));
	const fileChunks: TreeChange[][] = $derived(chunk(sortedChanges, 100));
	const visibleFiles: TreeChange[] = $derived(fileChunks.slice(0, currentDisplayIndex + 1).flat());

	const selectedFiles = $derived(idSelection.values(selectionId));

	const diffInputArgs = $derived<DiffInputContextArgs>({
		type: 'file-selection',
		projectId,
		selectedFiles,
		changes
	});

	const stackMacros = $derived(new StackMacros(projectId, stackService, uiState));
	const diffInputContext = $derived(
		new DiffInputContext(worktreeService, diffService, stackService, diffInputArgs)
	);
	const aiMacros = $derived(new AIMacros(projectId, aiService, promptService, diffInputContext));
	const aiGenEnabled = $derived(projectAiGenEnabled(projectId));

	$effect(() => {
		aiMacros.setGenAIEnabled($aiGenEnabled);
	});

	/**
	 * Create a branch and commit from the selected changes.
	 *
	 * _Branch [/bræntʃ/]_ is a verb that means to create a new branch and commit from the current changes.
	 *
	 * _According to who? Me._
	 *
	 * - Anonymous
	 */
	async function branchChanges() {
		const selectedFiles = idSelection.values(selectionId);
		if (selectionId.type !== 'worktree' || selectedFiles.length === 0) return;

		showToast({
			style: 'neutral',
			title: 'Creating a branch and commit...',
			message: 'This may take a few seconds.'
		});

		const selectedChanges: CreateCommitRequestWorktreeChanges[] = [];
		const treeChanges = changes.filter((change) =>
			selectedFiles.some((file) => file.path === change.path)
		);
		for (const file of selectedFiles) {
			const change = treeChanges.find((c) => c.path === file.path);
			if (!change) continue;
			const previousPathBytes = previousPathBytesFromTreeChange(change);
			selectedChanges.push({
				pathBytes: change.pathBytes,
				previousPathBytes,
				hunkHeaders: []
			});
		}

		const { branchName, commitMessage } = await aiMacros.getBranchNameAndCommitMessage();

		await stackMacros.branchChanges({
			worktreeChanges: selectedChanges,
			commitMessage,
			branchName
		});

		showToast({
			style: 'success',
			title: 'Branch and commit created successfully.',
			message: `Branch name: ${branchName}`
		});
	}

	function handleKeyDown(e: KeyboardEvent) {
		if (e.code === 'KeyB' && (e.ctrlKey || e.metaKey) && e.altKey) {
			branchChanges();
			e.preventDefault();
			return;
		}

		updateSelection({
			allowMultiple: true,
			metaKey: e.metaKey,
			shiftKey: e.shiftKey,
			key: e.key,
			targetElement: e.currentTarget as HTMLElement,
			files: sortedChanges,
			selectedFileIds: idSelection.values(selectionId),
			fileIdSelection: idSelection,
			selectionId: selectionId,
			preventDefault: () => e.preventDefault()
		});
	}

	function loadMore() {
		if (currentDisplayIndex + 1 >= fileChunks.length) return;
		currentDisplayIndex += 1;
	}

	const unrepresentedConflictedEntries = $derived.by(() => {
		if (!conflictEntries?.entries) return {};

		return Object.fromEntries(
			Object.entries(conflictEntries.entries).filter(([key, _value]) =>
				changes.every((change) => change.path !== key)
			)
		);
	});
</script>

{#snippet fileTemplate(change: TreeChange, idx: number, depth: number = 0)}
	{@const isExecutable = (change.status.subject as Modification).flags}
	<FileListItemWrapper
		{selectionId}
		{change}
		{projectId}
		{stackId}
		{active}
		{listMode}
		{depth}
		draggable={draggableFiles}
		executable={!!isExecutable}
		showCheckbox={showCheckboxes}
		isLast={idx === visibleFiles.length - 1}
		selected={idSelection.has(change.path, selectionId)}
		onclick={(e) => {
			selectFilesInList(e, change, sortedChanges, idSelection, true, idx, selectionId);
			onselect?.();
		}}
		{conflictEntries}
	/>
{/snippet}

{#each Object.entries(unrepresentedConflictedEntries) as [path, kind]}
	<FileListItemV3
		draggable={draggableFiles}
		filePath={path}
		conflicted
		conflictHint={conflictEntryHint(kind)}
		listMode="list"
	/>
{/each}
{#if visibleFiles.length > 0}
	{#if listMode === 'tree'}
		<!-- We don't need to use the `sortedChanges` here because
		`changeToFileTree` does the sorting for us -->
		{@const node = abbreviateFolders(changesToFileTree(changes))}
		<FileTreeNode isRoot {stackId} {node} {showCheckboxes} {changes} {fileTemplate} />
	{:else}
		<LazyloadContainer
			minTriggerCount={80}
			ontrigger={() => {
				loadMore();
			}}
			role="listbox"
			onkeydown={handleKeyDown}
		>
			{#each visibleFiles as change, idx}
				{@render fileTemplate(change, idx)}
			{/each}
		</LazyloadContainer>
	{/if}
{/if}
