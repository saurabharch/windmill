<script lang="ts">
	import { goto } from '$app/navigation'
	import { page } from '$app/stores'
	import { Alert, Badge, Drawer, DrawerContent, Tab, Tabs, UndoRedo } from '$lib/components/common'
	import Button from '$lib/components/common/button/Button.svelte'
	import DisplayResult from '$lib/components/DisplayResult.svelte'
	import FlowProgressBar from '$lib/components/flows/FlowProgressBar.svelte'
	import FlowStatusViewer from '$lib/components/FlowStatusViewer.svelte'
	import JobArgs from '$lib/components/JobArgs.svelte'
	import LogViewer from '$lib/components/LogViewer.svelte'
	import Path from '$lib/components/Path.svelte'
	import TestJobLoader from '$lib/components/TestJobLoader.svelte'
	import Toggle from '$lib/components/Toggle.svelte'
	import { AppService, DraftService, type Job, type Policy } from '$lib/gen'
	import { redo, undo } from '$lib/history'
	import { enterpriseLicense, workspaceStore } from '$lib/stores'
	import {
		AlignHorizontalSpaceAround,
		BellOff,
		Bug,
		Clipboard,
		DiffIcon,
		Expand,
		FileJson,
		FileUp,
		FormInput,
		History,
		Laptop2,
		Loader2,
		MoreVertical,
		RefreshCw,
		Save,
		Smartphone,
		FileClock
	} from 'lucide-svelte'
	import { getContext } from 'svelte'
	import { Pane, Splitpanes } from 'svelte-splitpanes'
	import {
		classNames,
		cleanValueProperties,
		copyToClipboard,
		truncateRev,
		orderedJsonStringify
	} from '../../../utils'
	import type {
		AppInput,
		ConnectedAppInput,
		RowAppInput,
		Runnable,
		StaticAppInput,
		UserAppInput
	} from '../inputType'
	import type { App, AppEditorContext, AppViewerContext } from '../types'
	import { BG_PREFIX, allItems, toStatic } from '../utils'
	import AppExportButton from './AppExportButton.svelte'
	import AppInputs from './AppInputs.svelte'
	import type { AppComponent } from './component/components'
	import PanelSection from './settingsPanel/common/PanelSection.svelte'
	import PreviewToggle from './PreviewToggle.svelte'

	import ToggleButtonGroup from '$lib/components/common/toggleButton-v2/ToggleButtonGroup.svelte'
	import ToggleButton from '$lib/components/common/toggleButton-v2/ToggleButton.svelte'
	import UnsavedConfirmationModal from '$lib/components/common/confirmationModal/UnsavedConfirmationModal.svelte'
	import Tooltip from '$lib/components/Tooltip.svelte'
	import { Sha256 } from '@aws-crypto/sha256-js'
	import { sendUserToast } from '$lib/toast'
	import DeploymentHistory from './DeploymentHistory.svelte'
	import Awareness from '$lib/components/Awareness.svelte'
	import { secondaryMenuLeftStore, secondaryMenuRightStore } from './settingsPanel/secondaryMenu'
	import ButtonDropdown from '$lib/components/common/button/ButtonDropdown.svelte'
	import { MenuItem } from '@rgossiaux/svelte-headlessui'
	import AppEditorTutorial from './AppEditorTutorial.svelte'
	import AppTimeline from './AppTimeline.svelte'
	import type DiffDrawer from '$lib/components/DiffDrawer.svelte'
	import AppReportsDrawer from './AppReportsDrawer.svelte'
	import HighlightCode from '$lib/components/HighlightCode.svelte'
	import { type ColumnDef, getPrimaryKeys } from '../components/display/dbtable/utils'
	import DebugPanel from './contextPanel/DebugPanel.svelte'
	import { getCountInput } from '../components/display/dbtable/queries/count'
	import { getSelectInput } from '../components/display/dbtable/queries/select'
	import { getInsertInput } from '../components/display/dbtable/queries/insert'
	import { getUpdateInput } from '../components/display/dbtable/queries/update'
	import { getDeleteInput } from '../components/display/dbtable/queries/delete'
	import { collectOneOfFields } from './appUtils'
	import Summary from '$lib/components/Summary.svelte'

	async function hash(message) {
		try {
			const msgUint8 = new TextEncoder().encode(message) // encode as (utf-8) Uint8Array
			const hashBuffer = await crypto.subtle.digest('SHA-256', msgUint8) // hash the message
			const hashArray = Array.from(new Uint8Array(hashBuffer)) // convert buffer to byte array
			const hashHex = hashArray.map((b) => b.toString(16).padStart(2, '0')).join('') // convert bytes to hex string
			return hashHex
		} catch {
			//subtle not available, trying pure js
			const hash = new Sha256()
			hash.update(message ?? '')
			const result = Array.from(await hash.digest())
			const hex = result.map((b) => b.toString(16).padStart(2, '0')).join('') // convert bytes to hex string
			return hex
		}
	}

	export let policy: Policy
	export let fromHub: boolean = false
	export let diffDrawer: DiffDrawer | undefined = undefined
	export let savedApp:
		| {
				value: App
				draft?: any
				path: string
				summary: string
				policy: any
				draft_only?: boolean
		  }
		| undefined = undefined
	export let version: number | undefined = undefined

	const {
		app,
		summary,
		breakpoint,
		appPath,
		jobs,
		jobsById,
		staticExporter,
		errorByComponent,
		openDebugRun
	} = getContext<AppViewerContext>('AppViewerContext')

	const { history, jobsDrawerOpen, refreshComponents } =
		getContext<AppEditorContext>('AppEditorContext')

	const loading = {
		publish: false,
		save: false,
		saveDraft: false
	}

	$: if ($openDebugRun == undefined) {
		$openDebugRun = (jobId: string) => {
			$jobsDrawerOpen = true
			selectedJobId = jobId
		}
	}

	let newPath: string = ''
	let pathError: string | undefined = undefined
	let appExport: AppExportButton

	let draftDrawerOpen = false
	let saveDrawerOpen = false
	let inputsDrawerOpen = fromHub
	let historyBrowserDrawerOpen = false
	let debugAppDrawerOpen = false
	let deploymentMsg: string | undefined = undefined

	function closeSaveDrawer() {
		saveDrawerOpen = false
	}

	function closeDraftDrawer() {
		draftDrawerOpen = false
	}

	function collectStaticFields(
		fields: Record<string, StaticAppInput | ConnectedAppInput | RowAppInput | UserAppInput>
	) {
		return Object.fromEntries(
			Object.entries(fields ?? {})
				.filter(([k, v]) => v.type == 'static')
				.map(([k, v]) => {
					return [k, v['value']]
				})
		)
	}

	type TriggerableV2 = {
		static_inputs: Record<string, any>
		one_of_inputs?: Record<string, any[] | undefined>
		allow_user_resources?: string[]
	}

	async function computeTriggerables() {
		const allTriggers: ([string, TriggerableV2] | undefined)[] = (await Promise.all(
			allItems($app.grid, $app.subgrids)
				.flatMap((x) => {
					let c = x.data as AppComponent
					let r: { input: AppInput | undefined; id: string }[] = [
						{ input: c.componentInput, id: x.id }
					]
					if (c.type === 'tablecomponent') {
						r.push(...c.actionButtons.map((x) => ({ input: x.componentInput, id: x.id })))
					}
					if (
						(c.type === 'aggridcomponent' ||
							c.type === 'aggridcomponentee' ||
							c.type === 'dbexplorercomponent' ||
							c.type === 'aggridinfinitecomponent' ||
							c.type === 'aggridinfinitecomponentee') &&
						Array.isArray(c.actions)
					) {
						r.push(...c.actions.map((x) => ({ input: x.componentInput, id: x.id })))
					}
					if (c.type === 'menucomponent') {
						r.push(...c.menuItems.map((x) => ({ input: x.componentInput, id: x.id })))
					}
					if (c.type === 'dbexplorercomponent') {
						let nr: { id: string; input: AppInput }[] = []
						let config = c.configuration as any

						const dbType = config?.type?.selected
						let pg = config?.type?.configuration?.[dbType]

						if (pg && dbType) {
							const { table, resource } = pg
							const tableValue = table.value
							const resourceValue = resource.value
							const columnDefs = (c.configuration.columnDefs as any).value as ColumnDef[]
							const whereClause = (c.configuration.whereClause as any).value as unknown as
								| string
								| undefined
							if (tableValue && resourceValue && columnDefs) {
								r.push({
									input: getSelectInput(resourceValue, tableValue, columnDefs, whereClause, dbType),
									id: x.id
								})

								r.push({
									input: getCountInput(resourceValue, tableValue, dbType, columnDefs, whereClause),
									id: x.id + '_count'
								})
								r.push({
									input: getInsertInput(tableValue, columnDefs, resourceValue, dbType),
									id: x.id + '_insert'
								})

								let primaryColumns = getPrimaryKeys(columnDefs)
								let columns = columnDefs?.filter((x) => primaryColumns.includes(x.field))

								r.push({
									input: getDeleteInput(resourceValue, tableValue, columns, dbType),
									id: x.id + '_delete'
								})

								columnDefs
									.filter((col) => col.editable || config.allEditable.value)
									.forEach((column) => {
										r.push({
											input: getUpdateInput(resourceValue, tableValue, column, columns, dbType),
											id: x.id + '_update'
										})
									})
							}
						}
						r.push(...nr)
					}

					const processed = r
						.filter((x) => x.input)
						.map(async (o) => {
							if (o.input?.type == 'runnable') {
								return await processRunnable(o.id, o.input.runnable, o.input.fields)
							}
						})

					return processed as Promise<[string, TriggerableV2] | undefined>[]
				})
				.concat(
					Object.values($app.hiddenInlineScripts ?? {}).map(async (v, i) => {
						return await processRunnable(BG_PREFIX + i, v, v.fields)
					}) as Promise<[string, TriggerableV2] | undefined>[]
				)
		)) as ([string, TriggerableV2] | undefined)[]

		delete policy.triggerables
		const ntriggerables: Record<string, TriggerableV2> = Object.fromEntries(
			allTriggers.filter(Boolean) as [string, TriggerableV2][]
		)
		policy.triggerables_v2 = ntriggerables
	}

	async function processRunnable(
		id: string,
		runnable: Runnable,
		fields: Record<string, any>
	): Promise<[string, TriggerableV2] | undefined> {
		const staticInputs = collectStaticFields(fields)
		const oneOfInputs = collectOneOfFields(fields, $app)
		const allowUserResources: string[] = Object.entries(fields)
			.map(([k, v]) => {
				return v['allowUserResources'] ? k : undefined
			})
			.filter(Boolean) as string[]

		if (runnable?.type == 'runnableByName') {
			let hex = await hash(runnable.inlineScript?.content)
			console.log('hex', hex, id)
			return [
				`${id}:rawscript/${hex}`,
				{
					static_inputs: staticInputs,
					one_of_inputs: oneOfInputs,
					allow_user_resources: allowUserResources
				}
			]
		} else if (runnable?.type == 'runnableByPath') {
			let prefix = runnable.runType !== 'hubscript' ? runnable.runType : 'script'
			return [
				`${id}:${prefix}/${runnable.path}`,
				{
					static_inputs: staticInputs,
					one_of_inputs: oneOfInputs,
					allow_user_resources: allowUserResources
				}
			]
		}
	}

	async function createApp(path: string) {
		await computeTriggerables()
		try {
			const appId = await AppService.createApp({
				workspace: $workspaceStore!,
				requestBody: {
					value: $app,
					path,
					summary: $summary,
					policy,
					deployment_message: deploymentMsg
				}
			})
			savedApp = {
				summary: $summary,
				value: structuredClone($app),
				path: path,
				policy: policy
			}
			closeSaveDrawer()
			sendUserToast('App deployed successfully')
			try {
				localStorage.removeItem(`app-${path}`)
			} catch (e) {
				console.error('error interacting with local storage', e)
			}
			goto(`/apps/edit/${appId}`)
		} catch (e) {
			sendUserToast('Error creating app', e)
		}
	}

	async function updateApp(npath: string) {
		await computeTriggerables()
		await AppService.updateApp({
			workspace: $workspaceStore!,
			path: appPath,
			requestBody: {
				value: $app!,
				summary: $summary,
				policy,
				path: npath,
				deployment_message: deploymentMsg
			}
		})
		savedApp = {
			summary: $summary,
			value: structuredClone($app),
			path: npath,
			policy
		}
		const appHistory = await AppService.getAppHistoryByPath({
			workspace: $workspaceStore!,
			path: npath
		})
		version = appHistory[0]?.version

		closeSaveDrawer()
		sendUserToast('App deployed successfully')
		if (appPath !== npath) {
			try {
				localStorage.removeItem(`app-${appPath}`)
			} catch (e) {
				console.error('error interacting with local storage', e)
			}
			window.location.pathname = `/apps/edit/${npath}?nodraft=true`
		}
	}

	let secretUrl: string | undefined = undefined

	$: appPath != '' && secretUrl == undefined && getSecretUrl()

	async function getSecretUrl() {
		secretUrl = await AppService.getPublicSecretOfApp({
			workspace: $workspaceStore!,
			path: appPath
		})
	}

	async function setPublishState() {
		await computeTriggerables()
		await AppService.updateApp({
			workspace: $workspaceStore!,
			path: appPath,
			requestBody: { policy }
		})
		if (policy.execution_mode == 'anonymous') {
			sendUserToast('App require no login to be accessed')
		} else {
			sendUserToast('App require login and read-access')
		}
	}

	async function save() {
		$secondaryMenuLeftStore.isOpen = false
		$secondaryMenuRightStore.isOpen = false

		saveDrawerOpen = true
		return
	}

	async function saveInitialDraft() {
		await computeTriggerables()
		try {
			await AppService.createApp({
				workspace: $workspaceStore!,
				requestBody: {
					value: $app,
					path: newPath,
					summary: $summary,
					policy,
					draft_only: true
				}
			})
			await DraftService.createDraft({
				workspace: $workspaceStore!,
				requestBody: {
					path: newPath,
					typ: 'app',
					value: {
						value: $app,
						path: newPath,
						summary: $summary,
						policy
					}
				}
			})
			savedApp = {
				summary: $summary,
				value: structuredClone($app),
				path: newPath,
				policy,
				draft_only: true,
				draft: {
					summary: $summary,
					value: structuredClone($app),
					path: newPath,
					policy
				}
			}

			draftDrawerOpen = false
			goto(`/apps/edit/${newPath}`)
		} catch (e) {
			sendUserToast('Error saving initial draft', e)
		}
		draftDrawerOpen = false
	}

	async function saveDraft(forceSave = false) {
		if ($page.params.path == undefined) {
			// initial draft
			draftDrawerOpen = true
			return
		}
		if (!savedApp) {
			return
		}
		const draftOrDeployed = cleanValueProperties(savedApp.draft || savedApp)
		const current = cleanValueProperties({
			summary: $summary,
			value: $app,
			path: newPath || savedApp.draft?.path || savedApp.path,
			policy
		})
		if (!forceSave && orderedJsonStringify(draftOrDeployed) === orderedJsonStringify(current)) {
			sendUserToast('No changes detected, ignoring', false, [
				{
					label: 'Save anyway',
					callback: () => {
						saveDraft(true)
					}
				}
			])
			return
		}
		loading.saveDraft = true
		try {
			await computeTriggerables()
			let path = $page.params.path
			if (savedApp.draft_only) {
				await AppService.deleteApp({
					workspace: $workspaceStore!,
					path: path
				})
				await AppService.createApp({
					workspace: $workspaceStore!,
					requestBody: {
						value: $app!,
						summary: $summary,
						policy,
						path: newPath || path,
						draft_only: true
					}
				})
			}
			await DraftService.createDraft({
				workspace: $workspaceStore!,
				requestBody: {
					path: savedApp.draft_only ? newPath || path : path,
					typ: 'app',
					value: {
						value: $app!,
						summary: $summary,
						policy,
						path: newPath || path
					}
				}
			})

			savedApp = {
				...(savedApp?.draft_only
					? {
							summary: $summary,
							value: structuredClone($app),
							path: savedApp.draft_only ? newPath || path : path,
							policy
					  }
					: savedApp),
				draft: {
					summary: $summary,
					value: structuredClone($app),
					path: newPath || path,
					policy
				}
			}

			sendUserToast('Draft saved')
			try {
				localStorage.removeItem(`app-${path}`)
			} catch (e) {
				console.error('error interacting with local storage', e)
			}
			loading.saveDraft = false
			if (newPath || path !== path) {
				goto(`/apps/edit/${newPath || path}`)
			}
		} catch (e) {
			loading.saveDraft = false
			throw e
		}
	}

	let onLatest = true
	async function compareVersions() {
		if (version === undefined) {
			return
		}
		const appHistory = await AppService.getAppHistoryByPath({
			workspace: $workspaceStore!,
			path: appPath
		})
		onLatest = version === appHistory[0]?.version
	}
	$: saveDrawerOpen && compareVersions()

	let selectedJobId: string | undefined = undefined
	let testJobLoader: TestJobLoader
	let job: Job | undefined = undefined
	let testIsLoading = false

	$: selectedJobId && !selectedJobId?.includes('Frontend') && testJobLoader?.watchJob(selectedJobId)

	$: if (selectedJobId?.includes('Frontend') && selectedJobId) {
		job = undefined
	}

	$: hasErrors = Object.keys($errorByComponent).length > 0

	let lock = false
	function onKeyDown(event: KeyboardEvent) {
		if (lock) return

		let classes = event.target?.['className']
		if (
			(typeof classes === 'string' && classes.includes('inputarea')) ||
			['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName!)
		) {
			return
		}

		lock = true

		switch (event.key) {
			case 'Z':
				if (event.ctrlKey || event.metaKey) {
					$app = redo(history)
					event.preventDefault()
				}
				break
			case 'z':
				if (event.ctrlKey || event.metaKey) {
					$app = undo(history, $app)

					event.preventDefault()
				}
				break
			case 's':
				if (event.ctrlKey || event.metaKey) {
					saveDraft()
					event.preventDefault()
				}
				break
			// case 'ArrowDown': {
			// 	let ids = generateIds()
			// 	let idx = ids.indexOf($selectedIdStore)
			// 	if (idx > -1 && idx < ids.length - 1) {
			// 		$selectedIdStore = ids[idx + 1]
			// 		event.preventDefault()
			// 	}
			// 	break
			// }
			// case 'ArrowUp': {
			// 	let ids = generateIds()
			// 	let idx = ids.indexOf($selectedIdStore)
			// 	if (idx > 0 && idx < ids.length) {
			// 		$selectedIdStore = ids[idx - 1]
			// 		event.preventDefault()
			// 	}
			// 	break
			// }
		}
		lock = false
	}

	let dirtyPath = false
	let path: Path | undefined = undefined

	let moreItems = [
		{
			displayName: 'Deployment History',
			icon: History,
			action: () => {
				historyBrowserDrawerOpen = true
			},
			disabled: !savedApp
		},
		{
			displayName: 'Export',
			icon: FileJson,
			action: () => {
				appExport.open($app)
			}
		},
		// {
		// 	displayName: 'Publish to Hub',
		// 	icon: faGlobe,
		// 	action: () => {
		// 		const url = appToHubUrl(toStatic($app, $staticExporter, $summary, $hubBaseUrlStore))
		// 		window.open(url.toString(), '_blank')
		// 	}
		// },
		{
			displayName: 'Hub compatible JSON',
			icon: FileUp,
			action: () => {
				appExport.open(toStatic($app, $staticExporter, $summary).app)
			}
		},
		{
			displayName: 'App Inputs',
			icon: FormInput,
			action: () => {
				inputsDrawerOpen = true
			}
		},
		{
			displayName: 'Schedule Reports',
			icon: FileClock,
			action: () => {
				appReportingDrawerOpen = true
			},
			disabled: !savedApp || savedApp.draft_only
		},
		{
			displayName: 'Diff',
			icon: DiffIcon,
			action: () => {
				if (!savedApp) {
					return
				}
				diffDrawer?.openDrawer()
				diffDrawer?.setDiff({
					mode: 'normal',
					deployed: savedApp,
					draft: savedApp.draft,
					current: {
						summary: $summary,
						value: $app,
						path: newPath || savedApp.draft?.path || savedApp.path,
						policy
					}
				})
			},
			disabled: !savedApp
		},
		// App debug menu
		{
			displayName: 'Troubleshoot panel',
			icon: Bug,
			action: () => {
				debugAppDrawerOpen = true
			}
		}
	]

	let appEditorTutorial: AppEditorTutorial | undefined = undefined

	export function toggleTutorial() {
		appEditorTutorial?.toggleTutorial()
	}

	let rightColumnSelect: 'timeline' | 'detail' = 'timeline'

	let appReportingDrawerOpen = false

	export function openTroubleshootPanel() {
		debugAppDrawerOpen = true
	}
</script>

<svelte:window on:keydown={onKeyDown} />

<TestJobLoader bind:this={testJobLoader} bind:isLoading={testIsLoading} bind:job />
<UnsavedConfirmationModal
	{diffDrawer}
	savedValue={savedApp}
	modifiedValue={{
		summary: $summary,
		value: $app,
		path: newPath || savedApp?.draft?.path || savedApp?.path,
		policy
	}}
/>

{#if appPath == ''}
	<Drawer bind:open={draftDrawerOpen} size="800px">
		<DrawerContent title="Initial draft save" on:close={() => closeDraftDrawer()}>
			<Alert title="Require path" type="info">
				Choose a path to save the initial draft of the app.
			</Alert>
			<h3>Summary</h3>
			<div class="w-full pt-2">
				<!-- svelte-ignore a11y-autofocus -->
				<input
					autofocus
					type="text"
					placeholder="App summary"
					class="text-sm w-full font-semibold"
					on:keydown|stopPropagation
					bind:value={$summary}
					on:keyup={() => {
						if (appPath == '' && $summary?.length > 0 && !dirtyPath) {
							path?.setName(
								$summary
									.toLowerCase()
									.replace(/[^a-z0-9_]/g, '_')
									.replace(/-+/g, '_')
									.replace(/^-|-$/g, '')
							)
						}
					}}
				/>
			</div>
			<div class="py-2" />
			<Path
				autofocus={false}
				bind:this={path}
				bind:error={pathError}
				bind:path={newPath}
				bind:dirty={dirtyPath}
				initialPath=""
				namePlaceholder="app"
				kind="app"
			/>
			<div class="py-4" />

			<div slot="actions">
				<Button
					startIcon={{ icon: Save }}
					disabled={pathError != ''}
					on:click={() => saveInitialDraft()}
				>
					Save initial draft
				</Button>
			</div>
		</DrawerContent>
	</Drawer>
{/if}
<Drawer bind:open={saveDrawerOpen} size="800px">
	<DrawerContent title="Deploy" on:close={() => closeSaveDrawer()}>
		{#if !onLatest}
			<Alert title="You're not on the latest app version" type="warning">
				By deploying, you may overwrite changes made by other users.
			</Alert>
			<div class="py-2" />
		{/if}
		<span class="text-secondary text-sm font-bold">Summary</span>
		<div class="w-full pt-2">
			<!-- svelte-ignore a11y-autofocus -->
			<input
				autofocus
				type="text"
				placeholder="App summary"
				class="text-sm w-full"
				bind:value={$summary}
				on:keydown|stopPropagation
				on:keyup={() => {
					if (appPath == '' && $summary?.length > 0 && !dirtyPath) {
						path?.setName(
							$summary
								.toLowerCase()
								.replace(/[^a-z0-9_]/g, '_')
								.replace(/-+/g, '_')
								.replace(/^-|-$/g, '')
						)
					}
				}}
			/>
		</div>
		<div class="py-4" />
		<span class="text-secondary text-sm font-bold">Deployment message</span>
		<div class="w-full pt-2">
			<!-- svelte-ignore a11y-autofocus -->
			<input
				type="text"
				placeholder="Optional deployment message"
				class="text-sm w-full"
				bind:value={deploymentMsg}
			/>
		</div>
		<div class="py-4" />
		<span class="text-secondary text-sm font-bold">Path</span>
		<Path
			bind:this={path}
			bind:dirty={dirtyPath}
			bind:error={pathError}
			bind:path={newPath}
			initialPath={appPath}
			namePlaceholder="app"
			kind="app"
			autofocus={false}
		/>

		<div slot="actions" class="flex flex-row gap-4">
			<Button
				variant="border"
				color="light"
				disabled={!savedApp || savedApp.draft_only}
				on:click={() => {
					if (!savedApp) {
						return
					}
					saveDrawerOpen = false
					diffDrawer?.openDrawer()
					diffDrawer?.setDiff({
						mode: 'normal',
						deployed: savedApp,
						draft: savedApp.draft,
						current: {
							summary: $summary,
							value: $app,
							path: newPath || savedApp.draft?.path || savedApp.path,
							policy
						},
						button: {
							text: 'Looks good, deploy',
							onClick: () => {
								if (appPath == '') {
									createApp(newPath)
								} else {
									updateApp(newPath)
								}
							}
						}
					})
				}}
			>
				<div class="flex flex-row gap-2 items-center">
					<DiffIcon size={14} />
					Diff
				</div>
			</Button>
			<Button
				startIcon={{ icon: Save }}
				disabled={pathError != ''}
				on:click={() => {
					if (appPath == '') {
						createApp(newPath)
					} else {
						updateApp(newPath)
					}
				}}
			>
				Deploy
			</Button>
		</div>
		<div class="py-2" />
		{#if appPath == ''}
			<Alert title="Require saving" type="error">
				Save this app once before you can publish it
			</Alert>
		{:else}
			<Alert title="App executed on behalf of you">
				A viewer of the app will execute the runnables of the app on behalf of the publisher (you)
				<Tooltip>
					It ensures that all required resources/runnable visible for publisher but not for viewer
					at time of creating the app would prevent the execution of the app. To guarantee tight
					security, a policy is computed at time of deployment of the app which only allow the
					scripts/flows referred to in the app to be called on behalf of. Furthermore, static
					parameters are not overridable. Hence, users will only be able to use the app as intended
					by the publisher without risk for leaking resources not used in the app.
				</Tooltip>
			</Alert>

			<div class="mt-10" />

			<h2>Secret public URL</h2>
			<div class="mt-4" />

			<Toggle
				options={{
					left: `Require login and read-access`,
					right: `No login required`
				}}
				checked={policy.execution_mode == 'anonymous'}
				on:change={(e) => {
					policy.execution_mode = e.detail ? 'anonymous' : 'publisher'
					setPublishState()
				}}
			/>

			<div class="my-6 box">
				Secret public url:
				{#if secretUrl}
					{@const url = `${$page.url.hostname}/public/${$workspaceStore}/${secretUrl}`}
					{@const href = $page.url.protocol + '//' + url}
					<a
						on:click={(e) => {
							e.preventDefault()
							copyToClipboard(href)
						}}
						{href}
						class="whitespace-nowrap text-ellipsis overflow-hidden mr-1 inline-flex gap-2"
					>
						{url}
						<span class="text-gray-700 ml-2">
							<Clipboard />
						</span>
					</a>
				{:else}<Loader2 class="animate-spin" />
				{/if}
				<div class="text-xs text-secondary"
					>You may share this url directly or embed it using an iframe (if not requiring login)</div
				>
			</div>

			<Alert type="info" title="Only latest deployed app is publicly available">
				You will still need to deploy the app to make visible the latest changes
			</Alert>
		{/if}
	</DrawerContent>
</Drawer>

<Drawer bind:open={inputsDrawerOpen} size="600px">
	<DrawerContent title="App inputs configuration" on:close={() => (inputsDrawerOpen = false)}>
		<AppInputs />
	</DrawerContent>
</Drawer>

<Drawer bind:open={historyBrowserDrawerOpen} size="1200px">
	<DrawerContent title="Deployment History" on:close={() => (historyBrowserDrawerOpen = false)}>
		<DeploymentHistory on:restore {appPath} />
	</DrawerContent>
</Drawer>

<Drawer bind:open={debugAppDrawerOpen} size="800px">
	<DrawerContent title="Troubleshoot Panel" on:close={() => (debugAppDrawerOpen = false)}>
		<DebugPanel />
	</DrawerContent>
</Drawer>

<Drawer bind:open={$jobsDrawerOpen} size="900px">
	<DrawerContent
		noPadding
		title="Debug Runs"
		on:close={() => {
			$jobsDrawerOpen = false
		}}
		tooltip="Look at latests runs to spot potential bugs."
		documentationLink="https://www.windmill.dev/docs/apps/app_debugging"
	>
		<Splitpanes class="!overflow-visible">
			<Pane size={25}>
				<PanelSection title="Past Runs">
					<div class="flex flex-col gap-2 w-full">
						{#if $jobs.length > 0}
							<div class="flex gap-2 flex-col-reverse">
								{#each $jobs ?? [] as id}
									{@const selectedJob = $jobsById[id]}
									{#if selectedJob}
										<!-- svelte-ignore a11y-click-events-have-key-events -->
										<!-- svelte-ignore a11y-no-static-element-interactions -->
										<div
											class={classNames(
												'border flex gap-1 truncate justify-between flex-row w-full items-center p-2 rounded-md cursor-pointer hover:bg-surface-secondary hover:text-blue-400',
												selectedJob.error ? 'border border-red-500 text-primary' : '',
												selectedJob.error && $errorByComponent[selectedJob.component]?.id == id
													? selectedJobId == id
														? 'bg-red-600 !border-blue-600'
														: 'bg-red-400'
													: selectedJobId == id
													? 'text-blue-600'
													: ''
											)}
											on:click={() => {
												selectedJobId = id
												rightColumnSelect = 'detail'
											}}
										>
											<span class="text-xs truncate">{truncateRev(selectedJob.job, 20)}</span>
											<Badge color="indigo">{selectedJob.component}</Badge>
										</div>
									{/if}
								{/each}
							</div>
						{:else}
							<div class="text-sm text-tertiary">No items</div>
						{/if}
					</div>
				</PanelSection>
			</Pane>
			<Pane size={75}>
				<div class="w-full h-full flex flex-col">
					<div>
						<Tabs bind:selected={rightColumnSelect}>
							<Tab value="timeline"><span class="font-semibold text-md">Timeline</span></Tab>
							<Tab value="detail"><span class="font-semibold">Details</span></Tab>
						</Tabs>
					</div>
					{#if rightColumnSelect == 'timeline'}
						<div class="p-2 grow overflow-auto">
							<AppTimeline />
						</div>
					{:else if rightColumnSelect == 'detail'}
						<div class="grow flex flex-col w-full overflow-auto">
							{#if selectedJobId}
								{#if selectedJobId?.includes('Frontend')}
									{@const jobResult = $jobsById[selectedJobId]}
									{#if jobResult?.error !== undefined}
										<Splitpanes horizontal class="grow border w-full">
											<Pane size={10} minSize={10}>
												<LogViewer
													content={`Logs are avaiable in the browser console directly`}
													isLoading={false}
													tag={undefined}
												/>
											</Pane>
											<Pane size={90} minSize={10} class="text-sm text-secondary">
												<div class="relative h-full px-2">
													<DisplayResult
														result={{
															error: { name: 'Frontend execution error', message: jobResult.error }
														}}
													/>
												</div>
											</Pane>
										</Splitpanes>
									{:else if jobResult !== undefined}
										<Splitpanes horizontal class="grow border w-full">
											<Pane size={10} minSize={10}>
												<LogViewer
													content={`Logs are avaiable in the browser console directly`}
													isLoading={false}
													tag={undefined}
												/>
											</Pane>
											<Pane size={90} minSize={10} class="text-sm text-secondary">
												<div class="relative h-full px-2">
													<DisplayResult
														workspaceId={$workspaceStore}
														jobId={selectedJobId}
														result={jobResult.result}
													/>
												</div>
											</Pane>
										</Splitpanes>
									{:else}
										<Loader2 class="animate-spin" />
									{/if}
								{:else}
									<div class="flex flex-col h-full w-full mb-4">
										{#if job?.['running']}
											<div class="flex flex-row-reverse w-full">
												<Button
													color="red"
													variant="border"
													on:click={() => testJobLoader?.cancelJob()}
												>
													<Loader2 size={14} class="animate-spin mr-2" />

													Cancel
												</Button>
											</div>
										{/if}
										{#if job?.args}
											<div class="p-2">
												<JobArgs
													id={job.id}
													workspace={job.workspace_id ?? $workspaceStore ?? 'no_w'}
													args={job?.args}
												/>
											</div>
										{/if}
										{#if job?.raw_code}
											<div class="pb-2 pl-2 pr-2 w-full overflow-auto h-full max-h-[80px]">
												<HighlightCode language={job?.language} code={job?.raw_code} />
											</div>
										{/if}

										{#if job?.job_kind !== 'flow' && job?.job_kind !== 'flowpreview'}
											{@const jobResult = $jobsById[selectedJobId]}
											<Splitpanes horizontal class="grow border w-full">
												<Pane size={50} minSize={10}>
													<LogViewer
														duration={job?.['duration_ms']}
														jobId={job?.id}
														content={job?.logs}
														isLoading={testIsLoading}
														tag={job?.tag}
													/>
												</Pane>
												<Pane size={50} minSize={10} class="text-sm text-secondary">
													{#if job != undefined && 'result' in job && job.result != undefined}
														<div class="relative h-full px-2">
															<DisplayResult
																workspaceId={$workspaceStore}
																jobId={selectedJobId}
																result={job.result}
															/></div
														>
													{:else if testIsLoading}
														<div class="p-2"><Loader2 class="animate-spin" /> </div>
													{:else if job != undefined && 'result' in job && job?.['result'] == undefined}
														<div class="p-2 text-tertiary">Result is undefined</div>
													{:else}
														<div class="p-2 text-tertiary">
															<Loader2 size={14} class="animate-spin mr-2" />
														</div>
													{/if}
												</Pane>
												{#if jobResult?.transformer}
													<Pane size={50} minSize={10} class="text-sm text-secondary p-2">
														<div class="font-bold">Transformer results</div>
														{#if job != undefined && 'result' in job && job.result != undefined}
															<div class="relative h-full px-2">
																<DisplayResult
																	workspaceId={$workspaceStore}
																	jobId={selectedJobId}
																	result={jobResult?.transformer}
																/>
															</div>
														{:else if testIsLoading}
															<div class="p-2"><Loader2 class="animate-spin" /> </div>
														{:else if job != undefined && 'result' in job && job?.['result'] == undefined}
															<div class="p-2 text-tertiary">Result is undefined</div>
														{:else}
															<div class="p-2 text-tertiary">
																<Loader2 size={14} class="animate-spin mr-2" />
															</div>
														{/if}
													</Pane>
												{/if}
											</Splitpanes>
										{:else}
											<div class="mt-10" />
											<FlowProgressBar {job} class="py-4" />
											<div class="w-full mt-10 mb-20">
												<FlowStatusViewer
													jobId={job.id}
													on:jobsLoaded={({ detail }) => {
														job = detail
													}}
												/>
											</div>
										{/if}
									</div>
								{/if}
							{:else}
								<div class="text-sm p-2 text-tertiary">Select a job to see its details</div>
							{/if}
						</div>
					{/if}
				</div>
			</Pane>
		</Splitpanes>
		<svelte:fragment slot="actions">
			<Button
				size="md"
				color="light"
				variant="border"
				on:click={() => {
					$refreshComponents?.()
				}}
				title="Refresh App"
			>
				Refresh app&nbsp;<RefreshCw size={16} />
			</Button>

			<Button
				size="md"
				color="light"
				variant="border"
				on:click={() => {
					errorByComponent.set({})
					jobs.set([])
				}}
				>Clear jobs
			</Button>
			{#if hasErrors}
				<Button size="md" color="light" variant="border" on:click={() => errorByComponent.set({})}
					>Clear Errors &nbsp;<BellOff size={14} />
				</Button>
			{/if}
		</svelte:fragment>
	</DrawerContent>
</Drawer>

<AppReportsDrawer bind:open={appReportingDrawerOpen} {appPath} />

<div
	class="border-b flex flex-row justify-between py-1 gap-2 gap-y-2 px-2 items-center overflow-y-visible overflow-x-auto"
>
	<div class="flex flex-row gap-2 items-center">
		<Summary bind:value={$summary} />
		<div class="flex gap-2">
			<UndoRedo
				undoProps={{ disabled: $history?.index === 0 }}
				redoProps={{ disabled: $history && $history?.index === $history.history.length - 1 }}
				on:undo={() => {
					$app = undo(history, $app)
				}}
				on:redo={() => {
					$app = redo(history)
				}}
			/>

			{#if $app}
				<ToggleButtonGroup class="h-[30px]" bind:selected={$app.fullscreen}>
					<ToggleButton
						icon={AlignHorizontalSpaceAround}
						value={false}
						tooltip="The max width is 1168px and the content stay centered instead of taking the full page width"
						iconProps={{ size: 16 }}
					/>
					<ToggleButton
						tooltip="The width is of the app if the full width of its container"
						icon={Expand}
						value={true}
						iconProps={{ size: 16 }}
					/>
				</ToggleButtonGroup>
			{/if}
			<div>
				<ToggleButtonGroup class="h-[30px]" bind:selected={$breakpoint}>
					<ToggleButton
						tooltip="Mobile View"
						icon={Smartphone}
						value="sm"
						iconProps={{ size: 16 }}
					/>
					<ToggleButton
						tooltip="Computer View"
						icon={Laptop2}
						value="lg"
						iconProps={{ size: 16 }}
					/>
				</ToggleButtonGroup>
			</div>
		</div>
	</div>

	{#if $enterpriseLicense && appPath != ''}
		<Awareness />
	{/if}
	<div class="flex flex-row gap-2 justify-end items-center overflow-visible">
		<ButtonDropdown hasPadding={false}>
			<svelte:fragment slot="buttonReplacement">
				<Button nonCaptureEvent size="xs" color="light">
					<div class="flex flex-row items-center">
						<MoreVertical size={14} />
					</div>
				</Button>
			</svelte:fragment>
			<svelte:fragment slot="items">
				{#each moreItems as item}
					<MenuItem
						on:click={item.action}
						disabled={item.disabled}
						class={item.disabled ? 'opacity-50' : ''}
					>
						<div
							class={classNames(
								'text-primary flex flex-row items-center text-left px-4 py-2 gap-2 cursor-pointer hover:bg-surface-hover !text-xs font-semibold'
							)}
						>
							<svelte:component this={item.icon} size={14} />
							{item.displayName}
						</div>
					</MenuItem>
				{/each}
			</svelte:fragment>
		</ButtonDropdown>
		<AppEditorTutorial bind:this={appEditorTutorial} />

		<div class="hidden md:inline relative overflow-visible">
			{#if hasErrors}
				<span
					class="animate-ping absolute inline-flex rounded-full bg-red-600 h-2 w-2 z-50 -right-0.5 -top-0.5"
				/>
				<span
					class=" absolute inline-flex rounded-full bg-red-600 h-2 w-2 z-50 -right-0.5 -top-0.5"
				/>
			{/if}
			<Button
				on:click={() => {
					if (selectedJobId == undefined && $jobs.length > 0) {
						selectedJobId = $jobs[$jobs.length - 1]
					}
					$jobsDrawerOpen = true
				}}
				color={hasErrors ? 'red' : 'light'}
				size="xs"
				variant="border"
				btnClasses="relative"
			>
				<div class="flex flex-row gap-1 items-center">
					<Bug size={14} />
					<div>Debug runs</div>
					<div class="text-2xs text-tertiary"
						>({$jobs?.length > 99 ? '99+' : $jobs?.length ?? 0})</div
					>
					{#if hasErrors}
						<Button
							size="xs"
							btnClasses="-my-2 !px-1 !py-0"
							color="light"
							variant="border"
							on:click={() => errorByComponent.set({})}
							><BellOff size={12} />
						</Button>
					{/if}
				</div>
			</Button>
		</div>
		<AppExportButton bind:this={appExport} />
		<PreviewToggle loading={loading.save} />
		<Button
			loading={loading.save}
			startIcon={{ icon: Save }}
			on:click={() => saveDraft()}
			size="xs"
			disabled={$page.params.path !== undefined && !savedApp}
			shortCut={{ key: 'S' }}
		>
			Draft
		</Button>
		<Button
			loading={loading.save}
			startIcon={{ icon: Save }}
			on:click={save}
			size="xs"
			dropdownItems={appPath != ''
				? () => [
						{
							label: 'Fork',
							onClick: () => {
								window.open(`/apps/add?template=${appPath}`)
							}
						}
				  ]
				: undefined}
		>
			Deploy
		</Button>
	</div>
</div>
