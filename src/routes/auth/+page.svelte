<script>
	import { toast } from 'svelte-sonner';

	import { onMount, getContext, tick } from 'svelte';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';

	import { getBackendConfig } from '$lib/apis';
	import { ldapUserSignIn, getSessionUser, userSignIn, userSignUp } from '$lib/apis/auths';

	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';
	import { WEBUI_NAME, config, user, socket } from '$lib/stores';

	import { generateInitialsImage, canvasPixelTest } from '$lib/utils';

	import Spinner from '$lib/components/common/Spinner.svelte';
	import OnBoarding from '$lib/components/OnBoarding.svelte';

	const i18n = getContext('i18n');

	let loaded = false;

	let mode = $config?.features.enable_ldap ? 'ldap' : 'signin';

	let name = '';
	let email = '';
	let password = '';

	let ldapUsername = '';

	const querystringValue = (key) => {
		const querystring = window.location.search;
		const urlParams = new URLSearchParams(querystring);
		return urlParams.get(key);
	};

	const setSessionUser = async (sessionUser) => {
		if (sessionUser) {
			console.log(sessionUser);
			toast.success($i18n.t(`You're now logged in.`));
			if (sessionUser.token) {
				localStorage.token = sessionUser.token;
			}
			$socket.emit('user-join', { auth: { token: sessionUser.token } });
			await user.set(sessionUser);
			await config.set(await getBackendConfig());

			const redirectPath = querystringValue('redirect') || '/';
			goto(redirectPath);
		}
	};

	const signInHandler = async () => {
		const sessionUser = await userSignIn(email, password).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		await setSessionUser(sessionUser);
	};

	const signUpHandler = async () => {
		const sessionUser = await userSignUp(name, email, password, generateInitialsImage(name)).catch(
			(error) => {
				toast.error(`${error}`);
				return null;
			}
		);

		await setSessionUser(sessionUser);
	};

	const ldapSignInHandler = async () => {
		const sessionUser = await ldapUserSignIn(ldapUsername, password).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
		await setSessionUser(sessionUser);
	};

	const submitHandler = async () => {
		if (mode === 'ldap') {
			await ldapSignInHandler();
		} else if (mode === 'signin') {
			await signInHandler();
		} else {
			await signUpHandler();
		}
	};

	const checkOauthCallback = async () => {
		if (!$page.url.hash) {
			return;
		}
		const hash = $page.url.hash.substring(1);
		if (!hash) {
			return;
		}
		const params = new URLSearchParams(hash);
		const token = params.get('token');
		if (!token) {
			return;
		}
		const sessionUser = await getSessionUser(token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
		if (!sessionUser) {
			return;
		}
		localStorage.token = token;
		await setSessionUser(sessionUser);
	};

	let onboarding = false;

	async function setLogoImage() {
		await tick();
		const logo = document.getElementById('logo');

		if (logo) {
			const isDarkMode = document.documentElement.classList.contains('dark');

			if (isDarkMode) {
				const darkImage = new Image();
				darkImage.src = '/static/favicon-dark.png';

				darkImage.onload = () => {
					logo.src = '/static/favicon-dark.png';
					logo.style.filter = ''; // Ensure no inversion is applied if favicon-dark.png exists
				};

				darkImage.onerror = () => {
					logo.style.filter = 'invert(1)'; // Invert image if favicon-dark.png is missing
				};
			}
		}
	}

	onMount(async () => {
		if ($user !== undefined) {
			const redirectPath = querystringValue('redirect') || '/';
			goto(redirectPath);
		}
		await checkOauthCallback();

		loaded = true;
		setLogoImage();

		if (($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false) {
			await signInHandler();
		} else {
			onboarding = $config?.onboarding ?? false;
		}
	});
</script>

<svelte:head>
	<title>
		H.E.L.P
	</title>
</svelte:head>

<OnBoarding
	bind:show={onboarding}
	getStartedHandler={() => {
		onboarding = false;
		mode = $config?.features.enable_ldap ? 'ldap' : 'signup';
	}}
/>

<div class="w-full h-screen max-h-[100dvh] text-black dark:text-white relative overflow-hidden">
	<!-- Background -->
	<div class="w-full h-full absolute top-0 left-0 bg-white dark:bg-gray-900"></div>

	<div class="w-full absolute top-0 left-0 right-0 h-8 drag-region" />

	{#if loaded}
		<!-- Logo -->
		<div class="fixed m-10 z-50">
			<div class="flex space-x-2">
				<div class="self-center">
					<div class="w-12 h-12 bg-gray-100 dark:bg-gray-800 rounded-2xl flex items-center justify-center border border-gray-200 dark:border-gray-700">
						<img
							id="logo"
							crossorigin="anonymous"
							src="{WEBUI_BASE_URL}/static/favicon.png"
							class="w-6 rounded-full"
							alt=""
						/>
					</div>
				</div>
			</div>
		</div>

		<div class="fixed bg-transparent min-h-screen w-full flex flex-col lg:flex-row font-primary z-50 text-black dark:text-white">
			<!-- Left side - Form -->
			<div class="w-full lg:w-1/2 min-h-screen flex items-center justify-center p-6">
				<div class="w-full max-w-md">
					{#if ($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false}
						<!-- Loading state -->
						<div class="w-full">
							<div class="bg-white dark:bg-gray-900 rounded-3xl p-12 shadow-lg border border-gray-200 dark:border-gray-700">
								<div class="flex items-center justify-center gap-3 text-xl sm:text-2xl text-center font-semibold text-gray-800 dark:text-gray-200">
									<div>
										{$i18n.t('Signing in to H.E.L.P')}
									</div>
									<div>
										<div class="w-6 h-6 border-2 border-gray-800 dark:border-gray-200 border-t-transparent rounded-full animate-spin"></div>
									</div>
								</div>
							</div>
						</div>
					{:else}
						<!-- Main auth form -->
						<div class="w-full">
							<div class="bg-white dark:bg-gray-900 rounded-3xl p-8 sm:p-10 shadow-lg border border-gray-200 dark:border-gray-700">
								<!-- Header -->
								<div class="mb-8 text-center">
									<h1 class="text-3xl font-bold text-gray-900 dark:text-white mb-2">
										{#if $config?.onboarding ?? false}
											{$i18n.t(`Get started with {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
										{:else if mode === 'ldap'}
											{$i18n.t(`Sign in with LDAP`)}
										{:else if mode === 'signin'}
											{$i18n.t(`Welcome back`)}
										{:else}
											{$i18n.t(`Create your account`)}
										{/if}
									</h1>
									<p class="text-gray-600 dark:text-gray-400">
										{#if $config?.onboarding ?? false}
											{$i18n.t('Create your admin account to get started')}
										{:else if mode === 'signin'}
											{$i18n.t('Sign in to continue to your account')}
										{:else}
											{$i18n.t('Join us today')}
										{/if}
									</p>
								</div>

								<!-- OAuth providers -->
								{#if Object.keys($config?.oauth?.providers ?? {}).length > 0}
									<div class="flex flex-col space-y-3 mb-6">
										{#if $config?.oauth?.providers?.google}
											<button
												class="w-full flex items-center justify-center px-4 py-3 border-2 border-gray-200 dark:border-gray-700 rounded-xl bg-white dark:bg-gray-800 text-gray-700 dark:text-gray-300 hover:bg-gray-50 dark:hover:bg-gray-700 transition-all duration-200 focus:outline-none focus:ring-2 focus:ring-gray-900 dark:focus:ring-white"
												on:click={() => {
													window.location.href = `${WEBUI_BASE_URL}/oauth/google/login`;
												}}
											>
												<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 48 48" class="w-5 h-5">
													<path fill="#EA4335" d="M24 9.5c3.54 0 6.71 1.22 9.21 3.6l6.85-6.85C35.9 2.38 30.47 0 24 0 14.62 0 6.51 5.38 2.56 13.22l7.98 6.19C12.43 13.72 17.74 9.5 24 9.5z"/>
													<path fill="#4285F4" d="M46.98 24.55c0-1.57-.15-3.09-.38-4.55H24v9.02h12.94c-.58 2.96-2.26 5.48-4.78 7.18l7.73 6c4.51-4.18 7.09-10.36 7.09-17.65z"/>
													<path fill="#FBBC05" d="M10.53 28.59c-.48-1.45-.76-2.99-.76-4.59s.27-3.14.76-4.59l-7.98-6.19C.92 16.46 0 20.12 0 24c0 3.88.92 7.54 2.56 10.78l7.97-6.19z"/>
													<path fill="#34A853" d="M24 48c6.48 0 11.93-2.13 15.89-5.81l-7.73-6c-2.15 1.45-4.92 2.3-8.16 2.3-6.26 0-11.57-4.22-13.47-9.91l-7.98 6.19C6.51 42.62 14.62 48 24 48z"/>
												</svg>
												<span class="ml-3 font-medium">{$i18n.t('Continue with Google')}</span>
											</button>
										{/if}

										{#if $config?.oauth?.providers?.github}
											<button
												class="w-full flex items-center justify-center px-4 py-3 border-2 border-gray-200 dark:border-gray-700 rounded-xl bg-white dark:bg-gray-800 text-gray-700 dark:text-gray-300 hover:bg-gray-50 dark:hover:bg-gray-700 transition-all duration-200 focus:outline-none focus:ring-2 focus:ring-gray-900 dark:focus:ring-white"
												on:click={() => {
													window.location.href = `${WEBUI_BASE_URL}/oauth/github/login`;
												}}
											>
												<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" class="w-5 h-5">
													<path fill="currentColor" d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.92 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57C20.565 21.795 24 17.31 24 12c0-6.63-5.37-12-12-12z"/>
												</svg>
												<span class="ml-3 font-medium">{$i18n.t('Continue with GitHub')}</span>
											</button>
										{/if}

										<div class="relative my-4">
											<div class="absolute inset-0 flex items-center">
												<div class="w-full border-t border-gray-200 dark:border-gray-700"></div>
											</div>
											<div class="relative flex justify-center">
												<span class="px-2 bg-white dark:bg-gray-900 text-sm text-gray-500 dark:text-gray-400">
													{$i18n.t('or')}
												</span>
											</div>
										</div>
									</div>
								{/if}

								<!-- Form fields -->
								{#if $config?.features.enable_login_form || $config?.features.enable_ldap}
									<form
										class="flex flex-col space-y-4"
										on:submit={(e) => {
											e.preventDefault();
											submitHandler();
										}}
									>
										{#if mode === 'signup'}
											<div>
												<label for="name" class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">
													{$i18n.t('Full Name')}
												</label>
												<input
													bind:value={name}
													type="text"
													id="name"
													class="w-full px-4 py-2.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-900 dark:focus:border-white focus:ring-2 focus:ring-gray-200 dark:focus:ring-gray-600 transition-all duration-200 outline-none"
													autocomplete="name"
													placeholder={$i18n.t('Enter your full name')}
													required
												/>
											</div>
										{/if}

										{#if mode === 'ldap'}
											<div>
												<label for="username" class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">
													{$i18n.t('Username')}
												</label>
												<input
													bind:value={ldapUsername}
													type="text"
													class="w-full px-4 py-2.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-900 dark:focus:border-white focus:ring-2 focus:ring-gray-200 dark:focus:ring-gray-600 transition-all duration-200 outline-none"
													autocomplete="username"
													name="username"
													id="username"
													placeholder={$i18n.t('Enter your username')}
													required
												/>
											</div>
										{:else}
											<div>
												<label for="email" class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">
													{$i18n.t('Email')}
												</label>
												<input
													bind:value={email}
													type="email"
													id="email"
													class="w-full px-4 py-2.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-900 dark:focus:border-white focus:ring-2 focus:ring-gray-200 dark:focus:ring-gray-600 transition-all duration-200 outline-none"
													autocomplete="email"
													name="email"
													placeholder={$i18n.t('Enter your email')}
													required
												/>
											</div>
										{/if}

										<div>
											<label for="password" class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">
												{$i18n.t('Password')}
											</label>
											<input
												bind:value={password}
												type="password"
												id="password"
												class="w-full px-4 py-2.5 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg text-gray-900 dark:text-white placeholder-gray-500 dark:placeholder-gray-400 focus:border-gray-900 dark:focus:border-white focus:ring-2 focus:ring-gray-200 dark:focus:ring-gray-600 transition-all duration-200 outline-none"
												placeholder={$i18n.t('Enter your password')}
												autocomplete="current-password"
												name="current-password"
												required
											/>
										</div>

										<!-- Submit button -->
										<div class="pt-2">
											<button
												class="w-full bg-gray-900 dark:bg-white text-white dark:text-gray-900 font-medium py-2.5 px-4 rounded-lg transition-all duration-200 hover:bg-gray-800 dark:hover:bg-gray-100 focus:outline-none focus:ring-2 focus:ring-gray-900 dark:focus:ring-white focus:ring-offset-2"
												type="submit"
											>
												{#if mode === 'ldap'}
													{$i18n.t('Sign in with LDAP')}
												{:else if mode === 'signin'}
													{$i18n.t('Sign in')}
												{:else if $config?.onboarding ?? false}
													{$i18n.t('Create Admin Account')}
												{:else}
													{$i18n.t('Create Account')}
												{/if}
											</button>
										</div>
									</form>

									<!-- Toggle between sign in/sign up -->
									{#if $config?.features.enable_signup && !($config?.onboarding ?? false)}
										<div class="mt-4 text-center text-sm">
											<span class="text-gray-600 dark:text-gray-400">
												{#if mode === 'signin'}
													{$i18n.t("Don't have an account?")}
												{:else}
													{$i18n.t('Already have an account?')}
												{/if}
											</span>
											<button
												class="ml-1.5 text-gray-900 dark:text-white font-medium hover:underline transition-all duration-200"
												type="button"
												on:click={() => {
													if (mode === 'signin') {
														mode = 'signup';
													} else {
														mode = 'signin';
													}
												}}
											>
												{#if mode === 'signin'}
													{$i18n.t('Sign up')}
												{:else}
													{$i18n.t('Sign in')}
												{/if}
											</button>
										</div>
									{/if}

									<!-- LDAP toggle -->
									{#if $config?.features.enable_ldap && $config?.features.enable_login_form}
										<div class="mt-4 text-center text-sm">
											<button
												class="text-gray-900 dark:text-white font-medium hover:underline transition-all duration-200"
												type="button"
												on:click={() => {
													if (mode === 'ldap')
														mode = ($config?.onboarding ?? false) ? 'signup' : 'signin';
													else mode = 'ldap';
												}}
											>
												{#if mode === 'ldap'}
													{$i18n.t('Continue with Email')}
												{:else}
													{$i18n.t('Continue with LDAP')}
												{/if}
											</button>
										</div>
									{/if}
								{/if}
							</div>
						</div>
					{/if}
				</div>
			</div>

			<!-- Right Side: Full Height Video -->
			<div class="hidden lg:flex lg:w-1/2 bg-gray-50 dark:bg-gray-800 items-center justify-center relative overflow-hidden">
				<div class="absolute inset-0 w-full h-full">
					<video 
						class="w-full h-full object-cover"
						autoplay 
						loop 
						muted 
						playsinline
						controlsList="nodownload" 
						disablePictureInPicture
					>
						<source src="http://localhost:5173/src/routes/auth/intro.mp4" type="video/mp4">
						Your browser does not support the video tag.
					</video>
				</div>
			</div>
		</div>
	{/if}
</div>

<style>
	:global(body) {
		font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
		@apply antialiased;
	}

	@media (max-width: 1024px) {
		.backdrop-blur-xl {
			backdrop-filter: blur(8px);
		}
	}
</style>