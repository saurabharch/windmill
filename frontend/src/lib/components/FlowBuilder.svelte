<script lang="ts">
	import {
		FlowService,
		ScheduleService,
		type Flow,
		type FlowModule,
		DraftService,
		type PathScript,
		ScriptService,
		type OpenFlow,
		type RawScript,
		type InputTransform
	} from '$lib/gen'
	import { initHistory, push, redo, undo } from '$lib/history'
	import {
		copilotInfo,
		enterpriseLicense,
		tutorialsToDo,
		userStore,
		workspaceStore
	} from '$lib/stores'
	import {
		cleanValueProperties,
		encodeState,
		formatCron,
		orderedJsonStringify,
		sleep
	} from '$lib/utils'
	import { sendUserToast } from '$lib/toast'
	import { Drawer } from '$lib/components/common'

	import { setContext, tick, type ComponentType } from 'svelte'
	import { writable, type Writable } from 'svelte/store'
	import CenteredPage from './CenteredPage.svelte'
	import { Badge, Button, UndoRedo } from './common'
	import FlowEditor from './flows/FlowEditor.svelte'
	import ScriptEditorDrawer from './flows/content/ScriptEditorDrawer.svelte'
	import type { FlowState } from './flows/flowState'
	import { dfs as dfsApply } from './flows/dfs'
	import { dfs, getPreviousIds } from './flows/previousResults'
	import FlowImportExportMenu from './flows/header/FlowImportExportMenu.svelte'
	import FlowPreviewButtons from './flows/header/FlowPreviewButtons.svelte'
	import { loadFlowSchedule, type Schedule } from './flows/scheduleUtils'
	import type { FlowEditorContext, FlowInput } from './flows/types'
	import { cleanInputs, emptyFlowModuleState } from './flows/utils'
	import {
		Calendar,
		Pen,
		Save,
		DiffIcon,
		MoreVertical,
		HistoryIcon,
		FileJson,
		type Icon,
		CornerDownLeft
	} from 'lucide-svelte'
	import { createEventDispatcher } from 'svelte'
	import Awareness from './Awareness.svelte'
	import { getAllModules } from './flows/flowExplorer'
	import {
		stepCopilot,
		type FlowCopilotModule,
		glueCopilot,
		type FlowCopilotContext
	} from './copilot/flow'
	import type { Schema, SchemaProperty } from '$lib/common'
	import FlowCopilotDrawer from './copilot/FlowCopilotDrawer.svelte'
	import FlowCopilotStatus from './copilot/FlowCopilotStatus.svelte'
	import { fade } from 'svelte/transition'
	import { loadFlowModuleState, pickScript } from './flows/flowStateUtils'
	import FlowCopilotInputsModal from './copilot/FlowCopilotInputsModal.svelte'
	import FlowBuilderTutorials from './FlowBuilderTutorials.svelte'

	import FlowTutorials from './FlowTutorials.svelte'
	import { ignoredTutorials } from './tutorials/ignoredTutorials'
	import type DiffDrawer from './DiffDrawer.svelte'
	import FlowHistory from './flows/FlowHistory.svelte'
	import ButtonDropdown from './common/button/ButtonDropdown.svelte'
	import { MenuItem } from '@rgossiaux/svelte-headlessui'
	import { twMerge } from 'tailwind-merge'
	import CustomPopover from './CustomPopover.svelte'
	import Summary from './Summary.svelte'

	export let initialPath: string = ''
	export let pathStoreInit: string | undefined = undefined
	export let newFlow: boolean
	export let selectedId: string | undefined
	export let initialArgs: Record<string, any> = {}
	export let loading = false
	export let flowStore: Writable<OpenFlow>
	export let flowStateStore: Writable<FlowState>
	export let savedFlow:
		| (Flow & {
				draft?: Flow | undefined
		  })
		| undefined = undefined
	export let diffDrawer: DiffDrawer | undefined = undefined

	const dispatch = createEventDispatcher()

	async function createSchedule(path: string) {
		const { cron, timezone, args, enabled, summary } = $scheduleStore

		try {
			await ScheduleService.createSchedule({
				workspace: $workspaceStore!,
				requestBody: {
					path: path,
					schedule: formatCron(cron),
					timezone,
					script_path: path,
					is_flow: true,
					args,
					enabled,
					summary
				}
			})
		} catch (err) {
			sendUserToast(`The primary schedule could not be created: ${err}`, true)
		}
	}

	let loadingSave = false
	let loadingDraft = false

	async function saveDraft(forceSave = false): Promise<void> {
		if (!newFlow && !savedFlow) {
			return
		}
		if (savedFlow) {
			const draftOrDeployed = cleanValueProperties(savedFlow.draft || savedFlow)
			const current = cleanValueProperties({ ...$flowStore, path: $pathStore })
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
		}
		loadingDraft = true
		try {
			const flow = cleanInputs($flowStore)
			try {
				localStorage.removeItem('flow')
				localStorage.removeItem(`flow-${$pathStore}`)
			} catch (e) {
				console.error('error interacting with local storage', e)
			}
			if (newFlow || savedFlow?.draft_only) {
				if (savedFlow?.draft_only) {
					await FlowService.deleteFlowByPath({
						workspace: $workspaceStore!,
						path: initialPath
					})
				}
				await FlowService.createFlow({
					workspace: $workspaceStore!,
					requestBody: {
						path: $pathStore,
						summary: flow.summary,
						description: flow.description ?? '',
						value: flow.value,
						schema: flow.schema,
						tag: flow.tag,
						draft_only: true,
						ws_error_handler_muted: flow.ws_error_handler_muted,
						visible_to_runner_only: flow.visible_to_runner_only
					}
				})
			}
			await DraftService.createDraft({
				workspace: $workspaceStore!,
				requestBody: {
					path: newFlow || savedFlow?.draft_only ? $pathStore : initialPath,
					typ: 'flow',
					value: {
						...flow,
						path: $pathStore
					}
				}
			})

			savedFlow = {
				...(newFlow || savedFlow?.draft_only
					? {
							...structuredClone($flowStore),
							path: $pathStore,
							draft_only: true
					  }
					: savedFlow),
				draft: {
					...structuredClone($flowStore),
					path: $pathStore
				}
			} as Flow & {
				draft?: Flow
			}

			if (newFlow) {
				dispatch('saveInitial', $pathStore)
			} else if (savedFlow?.draft_only && $pathStore !== initialPath) {
				initialPath = $pathStore
				// this is so we can use the flow builder outside of sveltekit
				dispatch('saveDraftOnlyAtNewPath', { path: $pathStore, selectedId: getSelectedId() })
			}
			sendUserToast('Saved as draft')
		} catch (error) {
			sendUserToast(`Error while saving the flow as a draft: ${error.body || error.message}`, true)
		}
		loadingDraft = false
	}

	export function computeUnlockedSteps(flow: Flow) {
		return Object.fromEntries(
			getAllModules(flow.value.modules, flow.value.failure_module)
				.filter((m) => m.value.type == 'script' && m.value.hash == null)
				.map((m) => [m.id, (m.value as PathScript).path])
		)
	}

	async function saveFlow(deploymentMsg?: string): Promise<void> {
		loadingSave = true
		try {
			const flow = cleanInputs($flowStore)
			// console.log('flow', computeUnlockedSteps(flow)) // del
			// loadingSave = false // del
			// return
			const { cron, timezone, args, enabled, summary } = $scheduleStore
			if (newFlow) {
				try {
					localStorage.removeItem('flow')
					localStorage.removeItem(`flow-${$pathStore}`)
				} catch (e) {
					console.error('error interacting with local storage', e)
				}
				await FlowService.createFlow({
					workspace: $workspaceStore!,
					requestBody: {
						path: $pathStore,
						summary: flow.summary,
						description: flow.description ?? '',
						value: flow.value,
						schema: flow.schema,
						ws_error_handler_muted: flow.ws_error_handler_muted,
						tag: flow.tag,
						dedicated_worker: flow.dedicated_worker,
						visible_to_runner_only: flow.visible_to_runner_only,
						deployment_message: deploymentMsg || undefined
					}
				})
				if (enabled) {
					await createSchedule($pathStore)
				}
			} else {
				try {
					localStorage.removeItem(`flow-${initialPath}`)
				} catch (e) {
					console.error('error interacting with local storage', e)
				}

				const scheduleExists = await ScheduleService.existsSchedule({
					workspace: $workspaceStore ?? '',
					path: initialPath
				})

				if (scheduleExists) {
					const schedule = await ScheduleService.getSchedule({
						workspace: $workspaceStore ?? '',
						path: initialPath
					})
					if (
						JSON.stringify(schedule.args) != JSON.stringify(args) ||
						schedule.schedule != cron ||
						schedule.timezone != timezone ||
						schedule.summary != summary
					) {
						await ScheduleService.updateSchedule({
							workspace: $workspaceStore ?? '',
							path: initialPath,
							requestBody: {
								schedule: formatCron(cron),
								timezone,
								args,
								summary
							}
						})
					}
					if (enabled != schedule.enabled) {
						await ScheduleService.setScheduleEnabled({
							workspace: $workspaceStore ?? '',
							path: initialPath,
							requestBody: { enabled }
						})
					}
				} else if (enabled) {
					await createSchedule(initialPath)
				}

				await FlowService.updateFlow({
					workspace: $workspaceStore!,
					path: initialPath,
					requestBody: {
						path: $pathStore,
						summary: flow.summary,
						description: flow.description ?? '',
						value: flow.value,
						schema: flow.schema,
						tag: flow.tag,
						dedicated_worker: flow.dedicated_worker,
						ws_error_handler_muted: flow.ws_error_handler_muted,
						visible_to_runner_only: flow.visible_to_runner_only,
						deployment_message: deploymentMsg || undefined
					}
				})
			}
			savedFlow = {
				...structuredClone($flowStore),
				path: $pathStore
			} as Flow
			loadingSave = false
			dispatch('deploy', $pathStore)
		} catch (err) {
			sendUserToast(`The flow could not be saved: ${err.body}`, true)
			loadingSave = false
		}
	}

	let timeout: NodeJS.Timeout | undefined = undefined

	$: {
		if ($flowStore || $selectedIdStore) {
			saveSessionDraft()
		}
	}

	function saveSessionDraft() {
		timeout && clearTimeout(timeout)
		timeout = setTimeout(() => {
			try {
				localStorage.setItem(
					initialPath && initialPath != '' ? `flow-${initialPath}` : 'flow',
					encodeState({
						flow: $flowStore,
						path: $pathStore,
						selectedId: $selectedIdStore
					})
				)
			} catch (err) {
				console.error(err)
			}
		}, 500)
	}

	const selectedIdStore = writable<string>(selectedId ?? 'settings-metadata')

	export function getSelectedId() {
		return $selectedIdStore
	}

	const scheduleStore = writable<Schedule>({
		summary: undefined,
		args: {},
		cron: '',
		timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
		enabled: false
	})
	const previewArgsStore = writable<Record<string, any>>(initialArgs)
	const scriptEditorDrawer = writable<ScriptEditorDrawer | undefined>(undefined)
	const moving = writable<{ module: FlowModule; modules: FlowModule[] } | undefined>(undefined)
	const history = initHistory($flowStore)
	const pathStore = writable<string>(pathStoreInit ?? initialPath)

	$: initialPath && ($pathStore = initialPath)

	const testStepStore = writable<Record<string, any>>({})

	function select(selectedId: string) {
		selectedIdStore.set(selectedId)
	}

	setContext<FlowEditorContext>('FlowEditorContext', {
		selectedId: selectedIdStore,
		schedule: scheduleStore,
		previewArgs: previewArgsStore,
		scriptEditorDrawer,
		moving,
		history,
		flowStateStore,
		flowStore,
		pathStore,
		testStepStore,
		saveDraft,
		initialPath,
		flowInputsStore: writable<FlowInput>({})
	})

	async function loadSchedule() {
		loadFlowSchedule(initialPath, $workspaceStore!)
			.then((schedule: Schedule) => {
				scheduleStore.set(schedule)
			})
			.catch(() => {
				scheduleStore.set({
					summary: undefined,
					cron: '0 */5 * * *',
					timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
					args: {},
					enabled: false
				})
			})
	}

	$: selectedId && select(selectedId)

	$: initialPath && initialPath != '' && $workspaceStore && loadSchedule()

	function onKeyDown(event: KeyboardEvent) {
		let classes = event.target?.['className']
		if (
			(typeof classes === 'string' && classes.includes('inputarea')) ||
			['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName!)
		) {
			return
		}

		switch (event.key) {
			case 'Z':
				if (event.ctrlKey || event.metaKey) {
					$flowStore = redo(history)
					event.preventDefault()
				}
				break
			case 'z':
				if (event.ctrlKey || event.metaKey) {
					$flowStore = undo(history, $flowStore)
					$selectedIdStore = 'Input'
					event.preventDefault()
				}
				break
			case 's':
				if (event.ctrlKey || event.metaKey) {
					saveDraft()
					event.preventDefault()
				}
				break
			case 'ArrowDown': {
				let ids = generateIds()
				let idx = ids.indexOf($selectedIdStore)
				if (idx > -1 && idx < ids.length - 1) {
					$selectedIdStore = ids[idx + 1]
					event.preventDefault()
				}
				break
			}
			case 'ArrowUp': {
				let ids = generateIds()
				let idx = ids.indexOf($selectedIdStore)
				if (idx > 0 && idx < ids.length) {
					$selectedIdStore = ids[idx - 1]
					event.preventDefault()
				}
				break
			}
		}
	}

	function generateIds() {
		return [
			'settings-metadata',
			'constants',
			...dfsApply($flowStore.value.modules, (module) => module.id)
		]
	}

	const dropdownItems: Array<{
		label: string
		onClick: () => void
	}> = []

	if (savedFlow?.draft_only === false || savedFlow?.draft_only === undefined) {
		dropdownItems.push({
			label: 'Exit & see details',
			onClick: () => dispatch('details', $pathStore)
		})
	}

	if (!newFlow) {
		dropdownItems.push({
			label: 'Fork',
			onClick: () => window.open(`/flows/add?template=${initialPath}`)
		})
	}

	let flowCopilotContext: FlowCopilotContext = {
		drawerStore: writable<Drawer | undefined>(undefined),
		modulesStore: writable<FlowCopilotModule[]>([]),
		currentStepStore: writable<string | undefined>(undefined),
		genFlow: undefined,
		shouldUpdatePropertyType: writable<{
			[key: string]: 'static' | 'javascript' | undefined
		}>({}),
		exprsToSet: writable<{
			[key: string]: InputTransform | any | undefined
		}>({}),
		generatedExprs: writable<{
			[key: string]: string | undefined
		}>({}),
		stepInputsLoading: writable<boolean>(false)
	}

	setContext('FlowCopilotContext', flowCopilotContext)

	const {
		drawerStore: copilotDrawerStore,
		modulesStore: copilotModulesStore,
		currentStepStore: copilotCurrentStepStore,
		shouldUpdatePropertyType
	} = flowCopilotContext

	let doneTs = 0
	async function getHubCompletions(text: string, idx: number, type: 'trigger' | 'script') {
		try {
			// make sure we display the results of the last request last
			const ts = Date.now()
			const scripts = (
				await ScriptService.queryHubScripts({
					text: `${text}`,
					limit: 3,
					kind: type
				})
			).map((s) => ({
				...s,
				path: `hub/${s.version_id}/${s.app}/${s.summary.toLowerCase().replaceAll(/\s+/g, '_')}`
			}))
			if (ts < doneTs) return
			doneTs = ts

			$copilotModulesStore[idx].hubCompletions = scripts as {
				path: string
				summary: string
				kind: string
				app: string
				ask_id: number
			}[]
		} catch (err) {
			if (err.name !== 'CancelError') throw err
		}
	}

	let abortController: AbortController | undefined = undefined
	let copilotLoading = false
	let flowCopilotMode: 'trigger' | 'sequence' = 'sequence'
	let copilotStatus: string = ''
	let copilotFlowInputs: Record<string, SchemaProperty> = {}
	let copilotFlowRequiredInputs: string[] = []
	let openCopilotInputsModal = false

	function setInitCopilotModules(mode: typeof flowCopilotMode) {
		$copilotModulesStore = [
			{
				id: 'a',
				type: mode === 'trigger' ? 'trigger' : 'script',
				description: '',
				code: '',
				hubCompletions: [],
				selectedCompletion: undefined,
				source: undefined,
				lang: undefined
			},
			{
				id: 'b',
				type: 'script',
				description: '',
				code: '',
				hubCompletions: [],
				selectedCompletion: undefined,
				source: undefined,
				lang: undefined
			}
		]
	}

	$: setInitCopilotModules(flowCopilotMode)

	function applyCopilotFlowInputs() {
		const properties = {
			...($flowStore.schema?.properties as Record<string, SchemaProperty> | undefined),
			...copilotFlowInputs
		}
		const required = [
			...(($flowStore.schema?.required as string[] | undefined) ?? []),
			...copilotFlowRequiredInputs
		]
		$flowStore.schema = {
			$schema: 'https://json-schema.org/draft/2020-12/schema',
			properties,
			required,
			type: 'object'
		}
	}

	function clearFlowInputsFromStep(id: string | undefined) {
		const module: FlowModule | undefined = dfs(id, $flowStore)[0]
		if (module?.value.type === 'rawscript' || module?.value.type === 'script') {
			// clear step inputs that start with flow_input. but not flow_input.iter
			for (const key in module.value.input_transforms) {
				const input = module.value.input_transforms[key]
				if (
					input.type === 'javascript' &&
					input.expr.includes('flow_input.') &&
					!input.expr.includes('flow_input.iter')
				) {
					module.value.input_transforms[key] = {
						type: 'static',
						value: undefined
					}
					$shouldUpdatePropertyType[key] = 'static'
				}
			}
		}
		$flowStore = $flowStore
	}

	async function finishStepGen() {
		copilotFlowInputs = {}
		copilotFlowRequiredInputs = []
		setInitCopilotModules(flowCopilotMode)
		copilotStatus = "Done! Just check the step's inputs and you're good to go!"
		await sleep(3000)
		copilotStatus = ''
	}

	function snakeCase(e: string): string {
		if (e.toLowerCase() === e) {
			return e
		}

		return (
			e
				.match(/([A-Z])/g)
				?.reduce((str, c) => str.replace(new RegExp(c), '_' + c.toLowerCase()), e)
				?.substring(e.slice(0, 1).match(/([A-Z])/g) ? 1 : 0) ?? e
		)
	}
	async function genFlow(idx: number, flowModules: FlowModule[], stepOnly = false) {
		try {
			push(history, $flowStore)
			let module = stepOnly ? $copilotModulesStore[0] : $copilotModulesStore[idx]

			copilotLoading = true
			copilotStatus = "Generating code for step '" + module.id + "'..."
			$copilotCurrentStepStore = module.id
			focusCopilot()

			if (!stepOnly && flowModules.length > idx) {
				select('')
				await tick()
				flowModules.splice(idx, flowModules.length - idx)
				$flowStore = $flowStore
				focusCopilot()
			}

			if (idx === 0 && !stepOnly) {
				$flowStore.schema = {
					$schema: 'https://json-schema.org/draft/2020-12/schema',
					properties: {},
					required: [],
					type: 'object'
				}
			}

			if (module.type === 'trigger') {
				if (!$scheduleStore.cron) {
					$scheduleStore.cron = '0 */15 * * *'
				}
				$scheduleStore.enabled = true
			}

			const flowModule: FlowModule & {
				value: RawScript | PathScript
			} = {
				id: module.id,
				stop_after_if:
					module.type === 'trigger'
						? {
								expr: 'result == undefined || Array.isArray(result) && result.length == 0',
								skip_if_stopped: true
						  }
						: undefined,
				value: {
					input_transforms: {},
					content: '',
					language: module.lang ?? 'bun',
					type: 'rawscript'
				},
				summary: module.description
			}

			let isHubStep = false
			if (module.source === 'hub' && module.selectedCompletion) {
				isHubStep = true
				const [hubScriptModule, hubScriptState] = await pickScript(
					module.selectedCompletion.path,
					`${module.selectedCompletion.summary} (${module.selectedCompletion.app})`,
					module.id,
					undefined
				)
				flowModule.value = hubScriptModule.value
				flowModule.summary = hubScriptModule.summary
				$flowStateStore[module.id] = hubScriptState
			} else {
				$flowStateStore[module.id] = emptyFlowModuleState()
			}

			if (stepOnly) {
				flowModules.splice(idx, 0, flowModule)
			} else if (idx === 1 && $copilotModulesStore[idx - 1].type === 'trigger') {
				const loopModule: FlowModule = {
					id: module.id + '_loop',
					value: {
						type: 'forloopflow',
						iterator: {
							type: 'javascript',
							expr: 'results.a'
						},
						skip_failures: true,
						modules: [flowModule]
					}
				}
				const loopState = await loadFlowModuleState(loopModule)
				$flowStateStore[loopModule.id] = loopState
				flowModules.push(loopModule)
			} else {
				flowModules.push(flowModule)
			}

			$copilotDrawerStore?.closeDrawer()
			await tick()
			select(module.id)
			await tick()
			await tick()
			focusCopilot()

			let isFirstInLoop = false
			const parents = dfs(module.id, $flowStore).slice(1)
			if (
				parents[0]?.value.type === 'forloopflow' &&
				parents[0].value.modules[0].id === module.id
			) {
				isFirstInLoop = true
			}
			const prevNodeId = getPreviousIds(module.id, $flowStore, false)[0] as string | undefined
			const pastModule = dfs(prevNodeId, $flowStore, false)[0] as FlowModule | undefined

			if (!module.source) {
				throw new Error('Invalid copilot module source')
			}

			if (module.source === 'custom') {
				const deltaStore = writable<string>('')
				const unsubscribe = deltaStore.subscribe(async (delta) => {
					module.editor?.append(delta)
				})

				abortController = new AbortController()
				await stepCopilot(
					module,
					deltaStore,
					$workspaceStore!,
					pastModule?.value.type === 'rawscript' || pastModule?.value.type === 'script'
						? (pastModule as FlowModule & {
								value: RawScript | PathScript
						  })
						: undefined,
					isFirstInLoop,
					abortController
				)
				unsubscribe()
			}

			copilotStatus = "Generating inputs for step '" + module.id + "'..."
			await sleep(500) // make sure code was parsed

			try {
				if (
					(flowModule.value.type === 'rawscript' || flowModule.value.type === 'script') &&
					(pastModule === undefined ||
						pastModule.value.type === 'rawscript' ||
						pastModule.value.type === 'script')
				) {
					const stepSchema: Schema = JSON.parse(JSON.stringify($flowStateStore[module.id].schema)) // deep copy
					if (isHubStep && pastModule !== undefined && $copilotInfo.exists_openai_resource_path) {
						// ask AI to set step inputs
						abortController = new AbortController()
						const { inputs, allExprs } = await glueCopilot(
							flowModule.value.input_transforms,
							$workspaceStore!,
							pastModule as FlowModule & {
								value: RawScript | PathScript
							},
							isFirstInLoop,
							abortController
						)

						// create flow inputs used by AI for autocompletion
						copilotFlowInputs = {}
						copilotFlowRequiredInputs = []
						Object.entries(allExprs).forEach(([key, expr]) => {
							if (expr.includes('flow_input.') && !expr.includes('flow_input.iter.')) {
								const flowInputKey = expr.match(/flow_input\.([A-Za-z0-9_]+)/)?.[1]
								if (
									flowInputKey !== undefined &&
									(!$flowStore.schema ||
										!(flowInputKey in (($flowStore.schema.properties as any) ?? {}))) // prevent overriding flow inputs
								) {
									if (key in stepSchema.properties) {
										copilotFlowInputs[flowInputKey] = stepSchema.properties[key]
										if (stepSchema.required.includes(key)) {
											copilotFlowRequiredInputs.push(flowInputKey)
										}
									} else {
										// when the key is nested (e.g. body.content)
										const [firstKey, ...rest] = key.split('.')
										const restKey = rest.join('.')
										const firstKeyProperties = stepSchema.properties[firstKey]?.properties
										if (firstKeyProperties !== undefined && restKey in firstKeyProperties) {
											copilotFlowInputs[flowInputKey] = firstKeyProperties[restKey]
											if (firstKeyProperties[restKey].required?.includes(flowInputKey)) {
												copilotFlowRequiredInputs.push(flowInputKey)
											}
										}
									}
								}
							}
						})

						if (!stepOnly) {
							applyCopilotFlowInputs()
						}

						// set step inputs
						Object.entries(inputs).forEach(([key, expr]) => {
							flowModule.value.input_transforms[key] = {
								type: 'javascript',
								expr
							}
							$shouldUpdatePropertyType[key] = 'javascript'
						})
					} else {
						if (
							isHubStep &&
							pastModule !== undefined &&
							!$copilotInfo.exists_openai_resource_path
						) {
							sendUserToast(
								'For better input generation, enable Windmill AI in the workspace settings',
								true
							)
						}

						// create possible flow inputs for autocompletion
						copilotFlowInputs = {}
						copilotFlowRequiredInputs = []
						Object.keys(flowModule.value.input_transforms).forEach((key) => {
							if (key !== 'prev_output') {
								const schema = $flowStateStore[module.id].schema
								const schemaProperty = Object.entries(schema.properties).find(
									(x) => x[0] === key
								)?.[1]
								const snakeKey = snakeCase(key)
								if (
									schemaProperty &&
									(!$flowStore.schema || !(snakeKey in ($flowStore.schema.properties as any) ?? {})) // prevent overriding flow inputs
								) {
									copilotFlowInputs[snakeKey] = schemaProperty
									if (schema.required.includes(snakeKey)) {
										copilotFlowRequiredInputs.push(snakeKey)
									}
								}
							}
						})
						if (!stepOnly) {
							applyCopilotFlowInputs()
						}

						// programatically set step inputs
						for (const key of Object.keys(flowModule.value.input_transforms)) {
							const snakeKey = snakeCase(key)
							flowModule.value.input_transforms[key] = {
								type: 'javascript',
								expr:
									key === 'prev_output'
										? isFirstInLoop
											? 'flow_input.iter.value'
											: pastModule
											? 'results.' + pastModule.id
											: 'flow_input.' + snakeKey
										: 'flow_input.' + snakeKey
							}
							$shouldUpdatePropertyType[key] = 'javascript'
						}
					}

					$flowStore = $flowStore // force rerendering
				} else {
					if (
						pastModule !== undefined &&
						pastModule.value.type !== 'rawscript' &&
						pastModule.value.type !== 'script'
					) {
						sendUserToast(
							`Linking to previous step ${pastModule.id} of type ${pastModule.value.type} is not yet supported`,
							true
						)
					} else {
						sendUserToast('Something went wrong, could not generate step inputs', true)
					}
				}
			} catch (err) {
				console.error(err)
			}

			if (stepOnly) {
				$copilotCurrentStepStore = undefined
				copilotLoading = false
				copilotStatus = ''
				if (Object.keys(copilotFlowInputs).length > 0) {
					openCopilotInputsModal = true
				} else {
					finishStepGen()
				}
			} else {
				copilotStatus =
					"Waiting for the user to validate code and inputs of step '" + module.id + "'"
			}
		} catch (err) {
			if (stepOnly) {
				copilotStatus = ''
				$copilotCurrentStepStore = undefined
				setInitCopilotModules(flowCopilotMode)
			}
			if (err?.message) {
				sendUserToast('Failed to generate code: ' + err.message, true)
			} else {
				sendUserToast('Failed to generate code', true)
				console.error(err)
			}
		} finally {
			copilotLoading = false
		}
	}

	flowCopilotContext.genFlow = genFlow

	async function finishCopilotFlowBuilder() {
		copilotLoading = true
		select('Input')
		$copilotCurrentStepStore = 'Input'

		copilotStatus = "Done! Just check the flow's inputs and you're good to go!"
		$copilotCurrentStepStore = undefined
		copilotLoading = false
		await sleep(3000)
		copilotStatus = ''
	}

	function focusCopilot() {
		document.querySelectorAll('.splitpanes__splitter').forEach((el) => {
			el.classList.add('hidden')
		})
		document.querySelectorAll('#flow-graph *').forEach((el) => {
			if (el instanceof HTMLElement) {
				el.style.pointerEvents = 'none'
			}
		})
	}

	function blurCopilot() {
		document.querySelectorAll('.splitpanes__splitter').forEach((el) => {
			el.classList.remove('hidden')
		})
		document.querySelectorAll('#flow-graph *').forEach((el) => {
			if (el instanceof HTMLElement) {
				el.style.pointerEvents = ''
			}
		})
	}

	$: $copilotCurrentStepStore === undefined && blurCopilot()

	let renderCount = 0
	let flowTutorials: FlowTutorials | undefined = undefined

	let jsonViewerDrawer: Drawer | undefined = undefined
	let flowHistory: FlowHistory | undefined = undefined

	export function triggerTutorial() {
		const urlParams = new URLSearchParams(window.location.search)
		const tutorial = urlParams.get('tutorial')

		if (tutorial) {
			flowTutorials?.runTutorialById(tutorial)
		} else if ($tutorialsToDo.includes(0) && !$ignoredTutorials.includes(0)) {
			flowTutorials?.runTutorialById('action')
		}
	}

	const moreItems: {
		displayName: string
		icon: ComponentType<Icon>
		action: () => void
		disabled?: boolean
	}[] = [
		{
			displayName: 'Deployment History',
			icon: HistoryIcon,
			action: () => {
				flowHistory?.open()
			},
			disabled: newFlow
		},
		{
			displayName: 'Export',
			icon: FileJson,
			action: () => jsonViewerDrawer?.openDrawer()
		}
	]

	let deploymentMsg = ''
	let msgInput: HTMLInputElement | undefined = undefined
</script>

<svelte:window on:keydown={onKeyDown} />

<slot />

{#key renderCount}
	{#if !$userStore?.operator}
		<FlowCopilotDrawer {getHubCompletions} {genFlow} bind:flowCopilotMode />
		{#if $pathStore}
			<FlowHistory bind:this={flowHistory} path={$pathStore} on:historyRestore />
		{/if}
		<FlowImportExportMenu bind:drawer={jsonViewerDrawer} />
		<FlowCopilotInputsModal
			on:confirmed={async () => {
				applyCopilotFlowInputs()
				finishStepGen()
			}}
			on:canceled={async () => {
				clearFlowInputsFromStep($copilotModulesStore[0]?.id)
				finishStepGen()
			}}
			bind:open={openCopilotInputsModal}
			inputs={Object.keys(copilotFlowInputs)}
		/>
		<ScriptEditorDrawer bind:this={$scriptEditorDrawer} />

		<div class="flex flex-col flex-1 h-screen">
			<!-- Nav between steps-->
			<div
				class="justify-between flex flex-row items-center pl-2.5 pr-6 space-x-4 scrollbar-hidden overflow-x-auto max-h-12 h-full relative"
			>
				{#if $copilotCurrentStepStore !== undefined}
					<div transition:fade class="absolute inset-0 bg-gray-500 bg-opacity-75 z-[900] !m-0" />
				{/if}
				<div class="flex w-full max-w-md gap-4 items-center">
					<Summary bind:value={$flowStore.summary} />

					<UndoRedo
						undoProps={{ disabled: $history.index === 0 }}
						redoProps={{ disabled: $history.index === $history.history.length - 1 }}
						on:undo={() => {
							const currentModules = $flowStore?.value?.modules

							$flowStore = undo(history, $flowStore)

							const newModules = $flowStore?.value?.modules
							const restoredModules = newModules?.filter(
								(node) => !currentModules?.some((currentNode) => currentNode?.id === node?.id)
							)

							for (const mod of restoredModules) {
								if (mod) {
									try {
										loadFlowModuleState(mod).then((state) => ($flowStateStore[mod.id] = state))
									} catch (e) {
										console.error('Error loading state for restored node', e)
									}
								}
							}

							$selectedIdStore = 'Input'
						}}
						on:redo={() => {
							$flowStore = redo(history)
						}}
					/>
				</div>

				<div class="gap-4 flex-row hidden md:flex w-full max-w-md">
					{#if $scheduleStore.enabled}
						<Button
							btnClasses="hidden lg:inline-flex"
							startIcon={{ icon: Calendar }}
							variant="contained"
							color="light"
							size="xs"
							on:click={async () => {
								select('settings-schedule')
							}}
						>
							{$scheduleStore.cron ?? ''}
						</Button>
					{/if}
					<div class="flex justify-start w-full">
						<div>
							<button
								on:click={async () => {
									select('settings-metadata')
									document.getElementById('path')?.focus()
								}}
							>
								<Badge
									color="gray"
									class="center-center !bg-gray-300 !text-tertiary dark:!bg-gray-700 dark:!text-gray-300 !h-[28px]  !w-[70px] rounded-r-none"
								>
									<Pen size={12} class="mr-2" /> Path
								</Badge>
							</button>
						</div>
						<input
							type="text"
							readonly
							value={$pathStore && $pathStore != '' ? $pathStore : 'Choose a path'}
							class="font-mono !text-xs !min-w-[96px] !max-w-[300px] !w-full !h-[28px] !my-0 !py-0 !border-l-0 !rounded-l-none"
							on:focus={({ currentTarget }) => {
								currentTarget.select()
							}}
						/>
					</div>
				</div>
				<div class="flex flex-row gap-2 items-center">
					{#if $enterpriseLicense && !newFlow}
						<Awareness />
					{/if}
					<div>
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
											class={twMerge(
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
					</div>

					<FlowBuilderTutorials
						on:reload={() => {
							renderCount += 1
						}}
					/>
					<Button
						color="light"
						variant="border"
						size="xs"
						on:click={() => {
							if (!savedFlow) {
								return
							}
							diffDrawer?.openDrawer()
							diffDrawer?.setDiff({
								mode: 'normal',
								deployed: savedFlow,
								draft: savedFlow['draft'],
								current: { ...$flowStore, path: $pathStore }
							})
						}}
						disabled={!savedFlow}
					>
						<div class="flex flex-row gap-2 items-center">
							<DiffIcon size={14} />
							Diff
						</div>
					</Button>

					<FlowCopilotStatus
						{copilotLoading}
						bind:copilotStatus
						{genFlow}
						{finishCopilotFlowBuilder}
						{abortController}
					/>

					<FlowPreviewButtons />
					<Button
						loading={loadingDraft}
						size="xs"
						startIcon={{ icon: Save }}
						on:click={() => saveDraft()}
						disabled={!newFlow && !savedFlow}
						shortCut={{
							key: 'S'
						}}
					>
						Draft
					</Button>

					<CustomPopover appearTimeout={0} focusEl={msgInput}>
						<Button
							loading={loadingSave}
							size="xs"
							startIcon={{ icon: Save }}
							on:click={() => saveFlow()}
							dropdownItems={!newFlow ? dropdownItems : undefined}
						>
							Deploy
						</Button>
						<svelte:fragment slot="overlay">
							<div class="flex flex-row gap-2 w-80">
								<input
									type="text"
									placeholder="Deployment message"
									bind:value={deploymentMsg}
									on:keydown={(e) => {
										if (e.key === 'Enter') {
											saveFlow(deploymentMsg)
										}
									}}
									bind:this={msgInput}
								/>
								<Button
									size="xs"
									on:click={() => saveFlow(deploymentMsg)}
									endIcon={{ icon: CornerDownLeft }}
									loading={loadingSave}
								>
									Deploy
								</Button>
							</div>
						</svelte:fragment>
					</CustomPopover>
				</div>
			</div>

			<!-- metadata -->
			{#if $flowStateStore}
				<FlowEditor
					{loading}
					on:reload={() => {
						renderCount += 1
					}}
				/>
			{:else}
				<CenteredPage>Loading...</CenteredPage>
			{/if}
		</div>
	{:else}
		Flow Builder not available to operators
	{/if}
{/key}

<FlowTutorials
	bind:this={flowTutorials}
	on:reload={() => {
		renderCount += 1
	}}
/>
