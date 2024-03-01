<script lang="ts">
	import { complianceColorMap } from './utils';
	import { page } from '$app/stores';
	import type { z } from 'zod';
	import type { SecurityFunctionSchema, ThreatSchema } from '$lib/utils/schemas';

	export let name: string;
	export let description: string;
	export let ra_id: string | undefined = undefined;
	export let leaf_content: string;
	export let threats: z.infer<typeof ThreatSchema>[] | undefined = undefined;
	export let security_functions: z.infer<typeof SecurityFunctionSchema>[] | undefined = undefined;
	export let children: Record<string, Record<string, unknown>> | undefined = undefined;
	export let canEditRequirementAssessment: boolean;
	export let status: string | undefined = undefined;
	export let statusCounts: Record<string, number> | undefined;
	export let assessable: boolean;

	const node = {
		name,
		description,
		ra_id,
		leaf_content,
		threats,
		security_functions,
		children,
		canEditRequirementAssessment,
		status,
		statusCounts,
		assessable
	} as const;

	type TreeViewItemNode = typeof node;

	$: hasChildren = children && Object.keys(children).length > 0;

	const content: string = leaf_content
		? leaf_content
		: description
		? `${name} ${description}`
		: name;

	let showInfo = false;

	const getAssessableNodes = (
		startNode: TreeViewItemNode,
		assessableNodes: TreeViewItemNode[] = []
	) => {
		if (startNode.assessable) assessableNodes.push(startNode);
		if (startNode.children) {
			for (const value of Object.values(startNode.children) as TreeViewItemNode[]) {
				getAssessableNodes(value, assessableNodes);
			}
		}
		return assessableNodes;
	};

	const assessableNodes = getAssessableNodes(node);

	const REQUIREMENT_ASSESSMENT_STATUS = [
		'compliant',
		'partially_compliant',
		'in_progress',
		'non_compliant',
		'not_applicable',
		'to_do'
	] as const;

	type StatusPercentage = {
		status: (typeof REQUIREMENT_ASSESSMENT_STATUS)[number];
		percentage: {
			value: number;
			display: string;
		};
	};

	const orderedStatusPercentages: StatusPercentage[] = REQUIREMENT_ASSESSMENT_STATUS.map(
		(status) => {
			if (!statusCounts) return { status, percentage: { value: 0, display: '0' } };
			const value = statusCounts[status] || 0;
			const percentValue: number = (value / assessableNodes.length) * 100;
			const percentage = {
				value: percentValue,
				display: percentValue.toFixed(0)
			};
			return { status, percentage };
		}
	);

	$: classesShowInfo = (show: boolean) => (!show ? 'hidden' : '');
	$: classesShowInfoText = (show: boolean) => (show ? 'text-primary-500' : '');
	$: classesPercentText = (statusColor: string) => (statusColor === '#000000' ? 'text-white' : '');
</script>

<div class="flex flex-row justify-between space-x-8">
	<div class="flex flex-1 max-w-[65ch] flex-col">
		<span style="font-weight: {hasChildren ? 600 : 300};">
			{#if assessable && canEditRequirementAssessment}
				<span class="w-full h-full flex rounded-token hover:text-primary-500">
					<a href="/requirement-assessments/{ra_id}?next={$page.url.pathname}">
						{content}
					</a>
				</span>
			{:else}
				<p class="max-w-[65ch] whitespace-pre-line">
					{content}
				</p>
			{/if}
		</span>
		{#if (threats && threats.length > 0) || (security_functions && security_functions.length > 0)}
			<div
				role="button"
				tabindex="0"
				class="select-none text-sm hover:text-primary-400 {classesShowInfoText(showInfo)}"
				on:click={(e) => {
					e.preventDefault();
					showInfo = !showInfo;
				}}
				on:keydown={(e) => {
					if (e.key === 'Enter') {
						e.preventDefault();
						showInfo = !showInfo;
					}
				}}
			>
				<i class="text-xs fa-solid fa-info-circle" /> Learn more
			</div>
			<div
				class="card p-2 variant-ghost-primary text-sm flex flex-row cursor-auto {classesShowInfo(
					showInfo
				)}"
			>
				<div class="flex-1">
					<p class="font-medium">
						<i class="fa-solid fa-gears" />
						Suggested security functions
					</p>
					{#if security_functions?.length === 0}
						<p>--</p>
					{:else if security_functions}
						<ul class="list-disc ml-4">
							{#each security_functions as func}
								<li>
									{#if func.id}
										<a
											class="anchor"
											href="/security-functions/{func.id}?next={$page.url.pathname}"
										>
											{func.name}
										</a>
									{:else}
										<p>{func.name}</p>
									{/if}
								</li>
							{/each}
						</ul>
					{/if}
				</div>
				<div class="flex-1">
					<p class="font-medium">
						<i class="fa-solid fa-gears" />
						Threats covered
					</p>
					{#if threats?.length === 0}
						<p>--</p>
					{:else if threats}
						<ul class="list-disc ml-4">
							{#each threats as threat}
								<li>
									{#if threat.id}
										<a class="anchor" href="/threats/{threat.id}?next={$page.url.pathname}">
											{threat.name}
										</a>
									{:else}
										<p>{threat.name}</p>
									{/if}
								</li>
							{/each}
						</ul>
					{/if}
				</div>
			</div>
		{/if}
	</div>
	{#if hasChildren}
		<div class="flex max-w-96 grow bg-gray-200 rounded-full overflow-hidden h-4 shrink self-center">
			{#each orderedStatusPercentages as sp}
				<div
					class="flex flex-col justify-center overflow-hidden text-xs text-center {classesPercentText(
						complianceColorMap[sp.status]
					)}"
					style="width: {sp.percentage.value}%; background-color: {complianceColorMap[sp.status]}"
				>
					{#if sp.status !== 'to_do'}
						{sp.percentage.display}%
					{/if}
				</div>
			{/each}
		</div>
	{/if}
</div>