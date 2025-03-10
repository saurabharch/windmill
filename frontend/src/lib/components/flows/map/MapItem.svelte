<script lang="ts">
	import { Button } from '$lib/components/common'
	import LanguageIcon from '$lib/components/common/languageIcons/LanguageIcon.svelte'
	import IconedResourceType from '$lib/components/IconedResourceType.svelte'
	import type { FlowModule } from '$lib/gen'
	import {
		Building,
		ClipboardCopy,
		GitBranchPlus,
		Repeat,
		Square,
		ArrowDown,
		GitBranch
	} from 'lucide-svelte'
	import { createEventDispatcher, getContext } from 'svelte'
	import type { Writable } from 'svelte/store'
	import FlowModuleSchemaItem from './FlowModuleSchemaItem.svelte'
	import InsertModuleButton from './InsertModuleButton.svelte'
	import { prettyLanguage } from '$lib/common'
	import { msToSec } from '$lib/utils'
	import BarsStaggered from '$lib/components/icons/BarsStaggered.svelte'
	import FlowJobsMenu from './FlowJobsMenu.svelte'

	export let mod: FlowModule
	export let trigger: boolean
	export let insertable: boolean
	export let insertableEnd = false
	export let annotation: string | undefined = undefined
	export let branchable: boolean = false
	export let bgColor: string = ''
	export let modules: FlowModule[]
	export let moving: string | undefined = undefined
	export let duration_ms: number | undefined = undefined
	export let disableAi: boolean = false
	export let wrapperId: string | undefined = undefined
	export let retries: number | undefined = undefined
	export let flowJobs:
		| { flowJobs: string[]; selected: number; flowJobsSuccess: (boolean | undefined)[] }
		| undefined

	$: idx = modules.findIndex((m) => m.id === mod.id)

	const { selectedId } = getContext<{ selectedId: Writable<string> }>('FlowGraphContext')
	const dispatch = createEventDispatcher<{
		delete: CustomEvent<MouseEvent>
		insert: {
			modules: FlowModule[]
			index: number
			detail: 'script' | 'forloop' | 'whileloop' | 'branchone' | 'branchall' | 'move'
			script?: { path: string; summary: string; hash: string | undefined }
		}
		select: string
		newBranch: { module: FlowModule }
		move: { module: FlowModule } | undefined
		selectedIteration: { index: number; id: string }
	}>()

	$: itemProps = {
		selected: $selectedId === mod.id,
		retry: mod.retry?.constant != undefined || mod.retry?.exponential != undefined,
		earlyStop: mod.stop_after_if != undefined,
		suspend: Boolean(mod.suspend),
		sleep: Boolean(mod.sleep),
		cache: Boolean(mod.cache_ttl),
		mock: Boolean(mod.mock?.enabled),
		concurrency: Boolean(mod?.value?.['concurrent_limit'])
	}

	function onDelete(event: CustomEvent<MouseEvent>) {
		dispatch('delete', event)
	}
	let openMenu: boolean | undefined = undefined
	let openMenu2: boolean | undefined = undefined
</script>

{#if mod}
	{#if insertable}
		<div
			class="{openMenu
				? 'z-20'
				: ''} w-[27px] absolute -top-[35px] left-[50%] right-[50%] -translate-x-1/2"
		>
			{#if moving}
				<button
					title="Add branch"
					on:click={() => {
						dispatch('insert', { modules, index: idx, detail: 'move' })
					}}
					type="button"
					disabled={wrapperId === moving}
					class=" text-primary bg-surface border mx-[1px] border-gray-300 dark:border-gray-500 focus:outline-none hover:bg-gray-100 focus:ring-4 focus:ring-gray-200 font-medium rounded-full text-sm w-[25px] h-[25px] flex items-center justify-center"
				>
					<ClipboardCopy class="m-[5px]" size={15} />
				</button>
			{:else}
				<InsertModuleButton
					{disableAi}
					bind:open={openMenu}
					{trigger}
					on:insert={(e) => {
						dispatch('insert', { modules, index: idx + 1, detail: 'script', script: e.detail })
					}}
					on:new={(e) => {
						dispatch('insert', { modules, index: idx, detail: e.detail })
					}}
					index={idx}
					{modules}
				/>
			{/if}
		</div>
	{/if}
	<div class="relative">
		{#if moving == mod.id}
			<div class="absolute z-10 right-20 top-0.5 center-center">
				<Button color="dark" on:click={() => dispatch('move')} size="xs" variant="border">
					Cancel move
				</Button>
			</div>
		{/if}

		{#if duration_ms}
			<div class="absolute z-10 right-0 -top-4 center-center text-tertiary text-2xs">
				{msToSec(duration_ms)}s
			</div>
		{/if}
		{#if annotation && annotation != ''}
			<div class="absolute z-10 left-0 -top-5 center-center text-tertiary">
				{annotation}
			</div>
		{/if}
		{#if flowJobs && !insertable}
			<div class="absolute z-10 right-8 -top-5">
				<FlowJobsMenu
					on:selectedIteration={(e) => {
						dispatch('selectedIteration', e.detail)
					}}
					flowJobsSuccess={flowJobs.flowJobsSuccess}
					flowJobs={flowJobs.flowJobs}
					selected={flowJobs.selected}
					index={idx}
				/>
			</div>
		{/if}

		<div class={moving == mod.id ? 'opacity-50' : ''}>
			{#if mod.value.type === 'forloopflow' || mod.value.type === 'whileloopflow'}
				<FlowModuleSchemaItem
					deletable={insertable}
					label={mod.summary ||
						`${mod.value.type == 'forloopflow' ? 'For' : 'While'} loop ${
							mod.value.parallel ? '(parallel)' : ''
						} ${mod.value.skip_failures ? '(skip failures)' : ''}`}
					id={mod.id}
					on:move={() => dispatch('move')}
					on:delete={onDelete}
					on:click={() => dispatch('select', mod.id)}
					{...itemProps}
					{bgColor}
					warningMessage={mod?.value?.type === 'forloopflow' &&
					mod?.value?.iterator?.type === 'javascript' &&
					mod?.value?.iterator?.expr === ''
						? 'Iterator expression is empty'
						: ''}
				>
					<div slot="icon">
						<Repeat size={16} />
					</div>
				</FlowModuleSchemaItem>
			{:else if mod.value.type === 'branchone'}
				<FlowModuleSchemaItem
					deletable={insertable}
					on:delete={onDelete}
					on:move={() => dispatch('move')}
					on:click={() => dispatch('select', mod.id)}
					{...itemProps}
					id={mod.id}
					label={mod.summary || 'Run one branch'}
					{bgColor}
				>
					<div slot="icon">
						<GitBranch size={16} />
					</div>
				</FlowModuleSchemaItem>
			{:else if mod.value.type === 'branchall'}
				<FlowModuleSchemaItem
					deletable={insertable}
					on:delete={onDelete}
					on:move={() => dispatch('move')}
					on:click={() => dispatch('select', mod.id)}
					id={mod.id}
					{...itemProps}
					label={mod.summary || `Run all branches${mod.value.parallel ? ' (parallel)' : ''}`}
					{bgColor}
				>
					<div slot="icon">
						<GitBranch size={16} />
					</div>
				</FlowModuleSchemaItem>
			{:else}
				<FlowModuleSchemaItem
					{retries}
					on:click={() => dispatch('select', mod.id)}
					on:delete={onDelete}
					on:move={() => dispatch('move')}
					deletable={insertable}
					id={mod.id}
					{...itemProps}
					modType={mod.value.type}
					{bgColor}
					label={mod.summary ||
						(`path` in mod.value ? mod.value.path : undefined) ||
						(mod.value.type === 'rawscript'
							? `Inline ${prettyLanguage(mod.value.language)}`
							: 'To be defined')}
				>
					<div slot="icon">
						{#if mod.value.type === 'rawscript'}
							<LanguageIcon lang={mod.value.language} width={16} height={16} />
						{:else if mod.summary == 'Terminate flow'}
							<Square size={16} />
						{:else if mod.value.type === 'identity'}
							<ArrowDown size={16} />
						{:else if mod.value.type === 'flow'}
							<BarsStaggered size={16} />
						{:else if mod.value.type === 'script'}
							{#if mod.value.path.startsWith('hub/')}
								<div>
									<IconedResourceType
										width="20px"
										height="20px"
										name={mod.value.path.split('/')[2]}
										silent={true}
									/>
								</div>
							{:else}
								<Building size={14} />
							{/if}
						{/if}
					</div>
				</FlowModuleSchemaItem>
			{/if}
		</div>
	</div>
	{#if insertable && insertableEnd}
		<div
			class="{openMenu2
				? 'z-20'
				: ''} w-[27px] absolute top-[49px] left-[50%] right-[50%] -translate-x-1/2"
		>
			{#if moving}
				<button
					title="Add branch"
					on:click={() => {
						dispatch('insert', { modules, index: idx + 1, detail: 'move' })
					}}
					type="button"
					disabled={wrapperId === moving}
					class=" text-primary bg-surface border mx-[1px] border-gray-300 dark:border-gray-500 focus:outline-none hover:bg-gray-100 focus:ring-4 focus:ring-gray-200 font-medium rounded-full text-sm w-[25px] h-[25px] flex items-center justify-center"
				>
					<ClipboardCopy class="m-[5px]" size={15} />
				</button>
			{:else}
				<InsertModuleButton
					{disableAi}
					bind:open={openMenu2}
					{trigger}
					on:insert={(e) => {
						dispatch('insert', { modules, index: idx + 1, detail: 'script', script: e.detail })
					}}
					on:new={(e) => {
						dispatch('insert', { modules, index: idx + 1, detail: e.detail })
					}}
					index={idx + 1}
					{modules}
				/>
			{/if}
		</div>
	{/if}

	{#if insertable && branchable}
		<div class="w-[27px] absolute top-[45px] left-[60%] right-[40%] -translate-x-1/2">
			<button
				title="Add branch"
				on:click={() => {
					dispatch('newBranch', { module: mod })
				}}
				type="button"
				id="add-branch-button"
				class=" text-primary bg-surface border mx-[1px] rotate-180 dark:border-gray-500 border-gray-300 focus:outline-none hover:bg-gray-100 focus:ring-4 focus:ring-gray-200 font-medium rounded-full text-sm w-[25px] h-[25px] flex items-center justify-center"
			>
				<GitBranchPlus class="m-[5px]" size={15} />
			</button>
		</div>
	{/if}
{/if}
