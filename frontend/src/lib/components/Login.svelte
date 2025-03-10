<script lang="ts">
	import { goto } from '$app/navigation'
	import Github from '$lib/components/icons/brands/Github.svelte'
	import Gitlab from '$lib/components/icons/brands/Gitlab.svelte'
	import Google from '$lib/components/icons/brands/Google.svelte'
	import Microsoft from '$lib/components/icons/brands/Microsoft.svelte'
	import Okta from '$lib/components/icons/brands/Okta.svelte'

	import { OauthService, UserService, WorkspaceService } from '$lib/gen'
	import { usersWorkspaceStore, workspaceStore, userStore } from '$lib/stores'
	import { classNames, emptyString, parseQueryParams } from '$lib/utils'
	import { getUserExt } from '$lib/user'
	import { Button, Skeleton } from '$lib/components/common'
	import { sendUserToast } from '$lib/toast'
	import { isCloudHosted } from '$lib/cloud'
	import { refreshSuperadmin } from '$lib/refreshUser'

	export let rd: string | undefined = undefined
	export let email: string | undefined = undefined
	export let password: string | undefined = undefined
	export let error: string | undefined = undefined

	const providers = [
		{
			type: 'github',
			name: 'GitHub',
			icon: Github
		},
		{
			type: 'gitlab',
			name: 'GitLab',
			icon: Gitlab
		},
		{
			type: 'google',
			name: 'Google',
			icon: Google
		},
		{
			type: 'microsoft',
			name: 'Microsoft',
			icon: Microsoft
		},
		{
			type: 'okta',
			name: 'Okta',
			icon: Okta
		}
	] as const

	const providersType = providers.map((p) => p.type as string)

	let showPassword = false
	let logins: string[] | undefined = undefined
	let saml: string | undefined = undefined

	async function login(): Promise<void> {
		if (!email || !password) {
			sendUserToast('Please fill in both email and password', true)
			return
		}

		const requestBody = {
			email,
			password
		}

		try {
			await UserService.login({ requestBody })
		} catch (err) {
			sendUserToast('Invalid credentials', true)
			return
		}

		// Once logged in, we can fetch the workspaces
		$usersWorkspaceStore = await WorkspaceService.listUserWorkspaces()
		// trigger a reload of the user
		if ($workspaceStore) {
			$userStore = await getUserExt($workspaceStore)
		}

		// Finally, we check whether the user is a superadmin
		refreshSuperadmin()
		redirectUser()
	}

	async function redirectUser() {
		const firstTimeCookie =
			document.cookie.match('(^|;)\\s*first_time\\s*=\\s*([^;]+)')?.pop() || '0'
		if (Number(firstTimeCookie) > 0 && email === 'admin@windmill.dev') {
			goto('/user/first-time')
			return
		}

		if (rd?.startsWith('http')) {
			window.location.href = rd
			return
		}
		if ($workspaceStore) {
			goto(rd ?? '/')
		} else {
			let workspaceTarget = parseQueryParams(rd ?? undefined)['workspace']
			if (rd && workspaceTarget) {
				$workspaceStore = workspaceTarget
				goto(rd)
				return
			}

			if (!$usersWorkspaceStore) {
				try {
					usersWorkspaceStore.set(await WorkspaceService.listUserWorkspaces())
				} catch {}
			}

			const allWorkspaces = $usersWorkspaceStore?.workspaces.filter((x) => x.id != 'admins')

			if (allWorkspaces?.length == 1) {
				workspaceStore.set(allWorkspaces[0].id)
				$userStore = await getUserExt($workspaceStore!)

				if (!$userStore?.is_super_admin && $userStore?.operator) {
					let defaultApp = await WorkspaceService.getWorkspaceDefaultApp({
						workspace: $workspaceStore!
					})
					if (!emptyString(defaultApp.default_app_path)) {
						goto(`/apps/get/${defaultApp.default_app_path}`)
					} else {
						goto(rd ?? '/')
					}
				} else {
					goto(rd ?? '/')
				}
			} else if (rd?.startsWith('/user/workspaces')) {
				goto(rd)
			} else if (rd == '/#user-settings') {
				goto(`/user/workspaces#user-settings`)
			} else {
				goto(`/user/workspaces${rd ? `?rd=${encodeURIComponent(rd)}` : ''}`)
			}
		}
	}

	async function loadLogins() {
		const allLogins = await OauthService.listOauthLogins()
		logins = allLogins.oauth
		saml = allLogins.saml

		showPassword = (logins.length == 0 && !saml) || (email != undefined && email.length > 0)
	}

	loadLogins()

	function handleKeyUp(event: KeyboardEvent) {
		const key = event.key

		if (key === 'Enter') {
			event.preventDefault()
			login()
		}
	}

	function storeRedirect(provider: string) {
		if (rd) {
			try {
				localStorage.setItem('rd', rd)
			} catch (e) {
				console.error('Could not persist redirection to local storage', e)
			}
		}
		window.location.href = window.location.origin + '/api/oauth/login/' + provider
	}

	$: error && sendUserToast(error, true)
</script>

<div class="bg-surface px-4 py-8 shadow md:border sm:rounded-lg sm:px-10">
	<div class="grid {logins && logins.length > 2 ? 'grid-cols-2' : ''} gap-4">
		{#if !logins}
			{#each Array(4) as _}
				<Skeleton layout={[0.5, [2.375]]} />
			{/each}
		{:else}
			{#each providers as { type, icon, name }}
				{#if logins?.includes(type)}
					<Button
						color="light"
						variant="border"
						startIcon={{ icon, classes: 'h-4' }}
						on:click={() => storeRedirect(type)}
					>
						{name}
					</Button>
				{/if}
			{/each}
			{#each logins.filter((x) => !providersType?.includes(x)) as login}
				<Button
					color="dark"
					variant="border"
					btnClasses="mt-2 w-full !border-gray-300"
					on:click={() => storeRedirect(login)}
				>
					{login}
				</Button>
			{/each}
		{/if}
		{#if saml}
			<Button
				color="dark"
				variant="border"
				btnClasses="mt-2 w-full !border-gray-300"
				on:click={() => {
					if (saml) {
						window.location.href = saml
					} else {
						sendUserToast('No SAML login available', true)
					}
				}}
			>
				SSO
			</Button>
		{/if}
	</div>
	{#if saml || (logins && logins.length > 0)}
		<div class={classNames('center-center', logins && logins.length > 0 ? 'mt-6' : '')}>
			<Button
				size="xs"
				color="blue"
				variant="border"
				btnClasses="!border-none"
				on:click={() => {
					showPassword = !showPassword
				}}
			>
				Log in without third-party
			</Button>
		</div>
	{/if}

	{#if showPassword}
		<div>
			<div class="space-y-6">
				{#if isCloudHosted()}
					<p class="text-xs text-tertiary italic pb-6">
						To get credentials without the OAuth providers above, send an email at
						contact@windmill.dev
					</p>
				{/if}
				<div>
					<label for="email" class="block text-sm font-medium leading-6 text-primary">
						Email
					</label>
					<div>
						<input
							type="email"
							bind:value={email}
							id="email"
							autocomplete="email"
							class="block w-full rounded-md border-0 py-1.5 text-primary shadow-sm ring-1 ring-inset placeholder:text-secondary focus:ring-2 focus:ring-inset focus:ring-frost-600 sm:text-sm sm:leading-6"
						/>
					</div>
				</div>

				<div>
					<label for="password" class="block text-sm font-medium leading-6 text-primary">
						Password
					</label>
					<div>
						<input
							on:keyup={handleKeyUp}
							bind:value={password}
							id="password"
							type="password"
							autocomplete="current-password"
							class="block w-full rounded-md border-0 py-1.5 text-shadow shadow-sm ring-1 ring-inset placeholder:text-secondary focus:ring-2 focus:ring-inset focus:ring-frost-600 sm:text-sm sm:leading-6"
						/>
					</div>
				</div>

				<div class="pt-2">
					<button
						on:click={login}
						disabled={!email || !password}
						class="flex w-full justify-center rounded-md bg-frost-600 px-3 py-2 text-sm font-semibold text-white shadow-sm hover:bg-frost-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-frost-600"
					>
						Sign in
					</button>
				</div>
			</div>

			{#if isCloudHosted()}
				<p class="text-2xs text-tertiary italic mt-10 text-center">
					By logging in, you agree to our
					<a href="https://windmill.dev/terms_of_service" target="_blank" rel="noreferrer">
						Terms of Service
					</a>
					and
					<a href="https://windmill.dev/privacy_policy" target="_blank" rel="noreferrer">
						Privacy Policy
					</a>
				</p>
			{/if}
		</div>
	{/if}
</div>
