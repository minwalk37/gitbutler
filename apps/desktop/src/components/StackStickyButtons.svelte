<script lang="ts">
	import type { Snippet } from 'svelte';

	type Props = {
		children: Snippet<[boolean]>;
	};

	let { children }: Props = $props();
	let isSticked = $state(true);
</script>

<div class="sticky-buttons" class:is-sticked={isSticked}>
	{#if children}
		{@render children(isSticked)}
	{/if}
</div>

<style lang="postcss">
	.sticky-buttons {
		display: flex;
		padding-top: 12px;
		padding-right: 12px;
		padding-bottom: 14px;
		padding-left: 12px;
		gap: 6px;
		border-top: 1px solid var(--clr-border-2);
		background-color: var(--clr-bg-1);
	}
</style>
