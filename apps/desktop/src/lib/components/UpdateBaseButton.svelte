<script lang="ts">
	import { showInfo, showError } from '$lib/notifications/toasts';
	import { getContext } from '$lib/utils/context';
	import { BranchController } from '$lib/vbranches/branchController';
	import Button from '@gitbutler/ui/Button.svelte';

	const branchController = getContext(BranchController);

	let loading = false;
</script>

<Button
	size="tag"
	style="error"
	kind="solid"
	help="Merge upstream commits into common base"
	onclick={async () => {
		loading = true;
		try {
			let infoText = await branchController.updateBaseBranch();
			if (infoText) {
				showInfo('Stashed conflicting branches', infoText);
			}
		} catch (err) {
			showError('Failed update workspace', err);
		} finally {
			loading = false;
		}
	}}
>
	{#if loading}
		busy...
	{:else}
		Update
	{/if}
</Button>
