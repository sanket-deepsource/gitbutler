<script lang="ts">
	import CommitAction from './CommitAction.svelte';
	import CommitCard from './CommitCard.svelte';
	import { Project } from '$lib/backend/projects';
	import { BaseBranch } from '$lib/baseBranch/baseBranch';
	import { transformAnyCommit } from '$lib/commitLines/transformers';
	import InsertEmptyCommitAction from '$lib/components/InsertEmptyCommitAction.svelte';
	import {
		ReorderDropzoneManagerFactory,
		type ReorderDropzone
	} from '$lib/dragging/reorderDropzoneManager';
	import Dropzone from '$lib/dropzone/Dropzone.svelte';
	import LineOverlay from '$lib/dropzone/LineOverlay.svelte';
	import { getGitHost } from '$lib/gitHost/interface/gitHost';
	import { getGitHostChecksMonitor } from '$lib/gitHost/interface/gitHostChecksMonitor';
	import { getGitHostListingService } from '$lib/gitHost/interface/gitHostListingService';
	import { getGitHostPrMonitor } from '$lib/gitHost/interface/gitHostPrMonitor';
	import { getContext } from '$lib/utils/context';
	import { getContextStore } from '$lib/utils/context';
	import { BranchController } from '$lib/vbranches/branchController';
	import {
		getIntegratedCommits,
		getLocalCommits,
		getLocalAndRemoteCommits,
		getRemoteCommits
	} from '$lib/vbranches/contexts';
	import { VirtualBranch } from '$lib/vbranches/types';
	import Button from '@gitbutler/ui/Button.svelte';
	import LineGroup from '@gitbutler/ui/commitLines/LineGroup.svelte';
	import { LineManagerFactory } from '@gitbutler/ui/commitLines/lineManager';
	import { goto } from '$app/navigation';

	export let isUnapplied: boolean;

	const branch = getContextStore(VirtualBranch);
	const localCommits = getLocalCommits();
	const localAndRemoteCommits = getLocalAndRemoteCommits();
	const remoteCommits = getRemoteCommits();
	const integratedCommits = getIntegratedCommits();
	const baseBranch = getContextStore(BaseBranch);
	const project = getContext(Project);
	const branchController = getContext(BranchController);
	const lineManagerFactory = getContext(LineManagerFactory);
	//
	const listingService = getGitHostListingService();
	const prMonitor = getGitHostPrMonitor();
	const checksMonitor = getGitHostChecksMonitor();

	const reorderDropzoneManagerFactory = getContext(ReorderDropzoneManagerFactory);
	const gitHost = getGitHost();

	$: mappedRemoteCommits =
		$remoteCommits.length > 0
			? [...$remoteCommits.map(transformAnyCommit), { id: 'remote-spacer' }]
			: [];
	$: mappedLocalCommits =
		$localCommits.length > 0
			? [...$localCommits.map(transformAnyCommit), { id: 'local-spacer' }]
			: [];

	$: lineManager = lineManagerFactory.build(
		{
			remoteCommits: mappedRemoteCommits,
			localCommits: mappedLocalCommits,
			localAndRemoteCommits: $localAndRemoteCommits.map(transformAnyCommit),
			integratedCommits: $integratedCommits.map(transformAnyCommit)
		},
		!isRebased
	);

	// Force the "base" commit lines to update when $branch updates.
	let tsKey: number | undefined;
	$: {
		$branch;
		tsKey = Date.now();
	}

	$: hasCommits = $branch.commits && $branch.commits.length > 0;
	$: headCommit = $branch.commits.at(0);

	$: hasRemoteCommits = $remoteCommits.length > 0;

	$: reorderDropzoneManager = reorderDropzoneManagerFactory.build($branch, [
		...$localCommits,
		...$localAndRemoteCommits
	]);

	$: forkPoint = $branch.forkPoint;
	$: upstreamForkPoint = $branch.upstreamData?.forkPoint;
	$: isRebased = !!forkPoint && !!upstreamForkPoint && forkPoint !== upstreamForkPoint;

	$: isPushingCommits = false;
	$: isIntegratingCommits = false;

	let baseIsUnfolded = false;

	function insertBlankCommit(commitId: string, location: 'above' | 'below' = 'below') {
		if (!$branch || !$baseBranch) {
			console.error('Unable to insert commit');
			return;
		}
		branchController.insertBlankCommit($branch.id, commitId, location === 'above' ? -1 : 1);
	}

	function getReorderDropzoneOffset({
		isFirst = false,
		isMiddle = false,
		isLast = false
	}: {
		isFirst?: boolean;
		isMiddle?: boolean;
		isLast?: boolean;
	}) {
		if (isFirst) return 12;
		if (isMiddle) return 6;
		if (isLast) return 0;
		return 0;
	}

	$: localCommitsConflicted = $localCommits.some((commit) => commit.conflicted);
</script>

{#snippet reorderDropzone(dropzone: ReorderDropzone, yOffsetPx: number)}
	<Dropzone accepts={dropzone.accepts.bind(dropzone)} ondrop={dropzone.onDrop.bind(dropzone)}>
		{#snippet overlay({ hovered, activated })}
			<LineOverlay {hovered} {activated} {yOffsetPx} />
		{/snippet}
	</Dropzone>
{/snippet}

{#if hasCommits || hasRemoteCommits}
	<div class="commits">
		<!-- UPSTREAM COMMITS -->

		{#if $remoteCommits.length > 0}
			<!-- To make the sticky position work, commits should be wrapped in a div -->
			<div class="commits-group">
				{#each $remoteCommits as commit, idx (commit.id)}
					<CommitCard
						type="remote"
						branch={$branch}
						{commit}
						{isUnapplied}
						first={idx === 0}
						last={idx === $remoteCommits.length - 1}
						commitUrl={$gitHost?.commitUrl(commit.id)}
						isHeadCommit={commit.id === headCommit?.id}
					>
						{#snippet lines(topHeightPx)}
							<LineGroup lineGroup={lineManager.get(commit.id)} {topHeightPx} />
						{/snippet}
					</CommitCard>
				{/each}

				<CommitAction>
					{#snippet lines()}
						<LineGroup lineGroup={lineManager.get('remote-spacer')} topHeightPx={0} />
					{/snippet}
					{#snippet action()}
						<Button
							style="warning"
							kind="solid"
							loading={isIntegratingCommits}
							onclick={async () => {
								isIntegratingCommits = true;
								try {
									await branchController.mergeUpstream($branch.id);
								} catch (e) {
									console.error(e);
								} finally {
									isIntegratingCommits = false;
								}
							}}>Integrate upstream</Button
						>
					{/snippet}
				</CommitAction>
			</div>
		{/if}

		<!-- LOCAL COMMITS -->
		{#if $localCommits.length > 0}
			<div class="commits-group">
				<InsertEmptyCommitAction
					isFirst
					on:click={() => insertBlankCommit($branch.head, 'above')}
				/>
				{@render reorderDropzone(
					reorderDropzoneManager.topDropzone,
					getReorderDropzoneOffset({ isFirst: true })
				)}
				{#each $localCommits as commit, idx (commit.id)}
					<CommitCard
						{commit}
						{isUnapplied}
						type="local"
						first={idx === 0}
						branch={$branch}
						last={idx === $localCommits.length - 1}
						isHeadCommit={commit.id === headCommit?.id}
					>
						{#snippet lines(topHeightPx)}
							<LineGroup lineGroup={lineManager.get(commit.id)} {topHeightPx} />
						{/snippet}
					</CommitCard>

					{@render reorderDropzone(
						reorderDropzoneManager.dropzoneBelowCommit(commit.id),
						getReorderDropzoneOffset({
							isLast: idx + 1 === $localCommits.length,
							isMiddle: idx + 1 === $localCommits.length
						})
					)}

					<InsertEmptyCommitAction
						isLast={idx + 1 === $localCommits.length}
						on:click={() => insertBlankCommit(commit.id, 'below')}
					/>
				{/each}

				<CommitAction bottomBorder={hasRemoteCommits}>
					{#snippet lines()}
						<LineGroup lineGroup={lineManager.get('local-spacer')} topHeightPx={0} />
					{/snippet}
					{#snippet action()}
						<Button
							style="pop"
							kind="solid"
							wide
							loading={isPushingCommits}
							disabled={localCommitsConflicted}
							help={localCommitsConflicted
								? 'In order to push, please resolve any conflicted commits.'
								: undefined}
							onclick={async () => {
								isPushingCommits = true;
								try {
									await branchController.pushBranch($branch.id, $branch.requiresForce);
									$listingService?.refresh();
									$prMonitor?.refresh();
									$checksMonitor?.update();
								} catch (e) {
									console.error(e);
								} finally {
									isPushingCommits = false;
								}
							}}
						>
							{$branch.requiresForce ? 'Force push' : 'Push'}
						</Button>
					{/snippet}
				</CommitAction>
			</div>
		{/if}

		<!-- LOCAL AND REMOTE COMMITS -->
		{#if $localAndRemoteCommits.length > 0}
			<div class="commits-group">
				{#each $localAndRemoteCommits as commit, idx (commit.id)}
					<CommitCard
						{commit}
						{isUnapplied}
						type="localAndRemote"
						first={idx === 0}
						branch={$branch}
						last={idx === $localAndRemoteCommits.length - 1}
						isHeadCommit={commit.id === headCommit?.id}
						commitUrl={$gitHost?.commitUrl(commit.id)}
					>
						{#snippet lines(topHeightPx)}
							<LineGroup lineGroup={lineManager.get(commit.id)} {topHeightPx} />
						{/snippet}
					</CommitCard>
					{@render reorderDropzone(
						reorderDropzoneManager.dropzoneBelowCommit(commit.id),
						getReorderDropzoneOffset({
							isMiddle: idx + 1 === $localAndRemoteCommits.length
						})
					)}
					<InsertEmptyCommitAction
						isLast={idx + 1 === $localAndRemoteCommits.length}
						on:click={() => insertBlankCommit(commit.id, 'below')}
					/>
				{/each}
			</div>
		{/if}

		<!-- INTEGRATED COMMITS -->
		{#if $integratedCommits.length > 0}
			<div class="commits-group">
				{#each $integratedCommits as commit, idx (commit.id)}
					<CommitCard
						{commit}
						{isUnapplied}
						type="integrated"
						first={idx === 0}
						branch={$branch}
						isHeadCommit={commit.id === headCommit?.id}
						last={idx === $integratedCommits.length - 1}
						commitUrl={$gitHost?.commitUrl(commit.id)}
					>
						{#snippet lines(topHeightPx)}
							<LineGroup lineGroup={lineManager.get(commit.id)} {topHeightPx} />
						{/snippet}
					</CommitCard>
				{/each}
			</div>
		{/if}

		<!-- BASE -->
		<div class="base-row-container" class:base-row-container_unfolded={baseIsUnfolded}>
			<div
				class="base-row"
				tabindex="0"
				role="button"
				on:click|stopPropagation={() => (baseIsUnfolded = !baseIsUnfolded)}
				on:keydown={(e) => e.key === 'Enter' && (baseIsUnfolded = !baseIsUnfolded)}
			>
				<div class="base-row__lines">
					{#key tsKey}
						<LineGroup lineGroup={lineManager.base} />
					{/key}
				</div>
				<div class="base-row__content">
					<span class="text-11 base-row__text"
						>Base commit <button
							class="base-row__commit-link"
							on:click={async () => await goto(`/${project.id}/base`)}
						>
							{$branch.forkPoint ? $branch.forkPoint.slice(0, 7) : ''}
						</button>
					</span>
				</div>
			</div>
		</div>
	</div>
{/if}

<style lang="postcss">
	.commits {
		position: relative;
		display: flex;
		flex-direction: column;
		background-color: var(--clr-bg-2);
		border-top: 1px solid var(--clr-border-2);
		border-bottom-left-radius: var(--radius-m);
		border-bottom-right-radius: var(--radius-m);

		--base-top-margin: 8px;
		--base-row-height: 20px;
		--base-row-height-unfolded: 48px;

		--base-icon-top: -8px;

		--avatar-first-top: 50px;
		--avatar-top: 16px;
	}

	/* BASE ROW */

	.base-row-container {
		display: flex;
		flex-direction: column;
		height: var(--base-row-height);
		border-bottom-left-radius: var(--radius-m);
		border-bottom-right-radius: var(--radius-m);

		overflow: hidden;
		transition: height var(--transition-medium);

		&:hover {
			&:not(.base-row-container_unfolded) {
				height: 22px;
			}

			& .base-row {
				background-color: var(--clr-bg-2-muted);
			}
		}
	}

	.base-row-container_unfolded {
		--base-row-height: var(--base-row-height-unfolded);
		--base-icon-top: -3px;

		& .base-row__text {
			opacity: 1;
		}
	}

	.base-row {
		display: flex;
		gap: 8px;
		border-top: 1px solid var(--clr-border-3);
		min-height: calc(var(--base-row-height-unfolded) - var(--base-top-margin));
		margin-top: var(--base-top-margin);
		transition: background-color var(--transition-fast);
	}

	.base-row__lines {
		display: flex;
		margin-top: -9px;
	}

	.base-row__content {
		display: flex;
		align-items: center;
	}

	.base-row__text {
		color: var(--clr-text-2);
		opacity: 0;
		margin-top: 2px;
		transition: opacity var(--transition-medium);
	}

	.base-row__commit-link {
		text-decoration: underline;
		cursor: pointer;
	}
</style>
