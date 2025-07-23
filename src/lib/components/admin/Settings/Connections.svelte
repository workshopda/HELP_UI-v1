<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { createEventDispatcher, onMount, getContext } from 'svelte';

	const dispatch = createEventDispatcher();

	import { getOllamaConfig, updateOllamaConfig } from '$lib/apis/ollama';
	import { getOpenAIConfig, updateOpenAIConfig, getOpenAIModels } from '$lib/apis/openai';
	import { getModels as _getModels } from '$lib/apis';
	import { getDirectConnectionsConfig, setDirectConnectionsConfig } from '$lib/apis/configs';

	import { config, models, settings, user } from '$lib/stores';

	import Switch from '$lib/components/common/Switch.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Plus from '$lib/components/icons/Plus.svelte';

	import OpenAIConnection from './Connections/OpenAIConnection.svelte';
	import AddConnectionModal from '$lib/components/AddConnectionModal.svelte';
	import OllamaConnection from './Connections/OllamaConnection.svelte';

	const i18n = getContext('i18n');

	interface ApiConfig {
		[key: string]: any;
	}

	let OLLAMA_BASE_URLS: string[] = [''];
	let OLLAMA_API_CONFIGS: {[key: number]: ApiConfig} = {};

	let OPENAI_API_KEYS: string[] = [''];
	let OPENAI_API_BASE_URLS: string[] = [''];
	let OPENAI_API_CONFIGS: {[key: number]: ApiConfig} = {};

	let ENABLE_OPENAI_API: boolean | null = null;
	let ENABLE_OLLAMA_API: boolean | null = null;

	let directConnectionsConfig: {ENABLE_DIRECT_CONNECTIONS?: boolean} | null = null;
	let isLoading = true;

	let pipelineUrls: {[key: string]: boolean} = {};
	let showAddOpenAIConnectionModal = false;
	let showAddOllamaConnectionModal = false;

	// Computed property for direct connections switch
	$: directConnectionsEnabled = directConnectionsConfig?.ENABLE_DIRECT_CONNECTIONS ?? false;

	const getModels = async () => {
		const models = await _getModels(
			localStorage.token,
			$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
		);
		return models;
	};

	const updateOpenAIHandler = async () => {
		if (ENABLE_OPENAI_API !== null) {
			OPENAI_API_BASE_URLS = OPENAI_API_BASE_URLS.map((url) => url.replace(/\/$/, ''));

			if (OPENAI_API_KEYS.length !== OPENAI_API_BASE_URLS.length) {
				if (OPENAI_API_KEYS.length > OPENAI_API_BASE_URLS.length) {
					OPENAI_API_KEYS = OPENAI_API_KEYS.slice(0, OPENAI_API_BASE_URLS.length);
				}

				if (OPENAI_API_KEYS.length < OPENAI_API_BASE_URLS.length) {
					const diff = OPENAI_API_BASE_URLS.length - OPENAI_API_KEYS.length;
					for (let i = 0; i < diff; i++) {
						OPENAI_API_KEYS.push('');
					}
				}
			}

			try {
				const res = await updateOpenAIConfig(localStorage.token, {
					ENABLE_OPENAI_API: ENABLE_OPENAI_API,
					OPENAI_API_BASE_URLS: OPENAI_API_BASE_URLS,
					OPENAI_API_KEYS: OPENAI_API_KEYS,
					OPENAI_API_CONFIGS: OPENAI_API_CONFIGS
				});

				if (res) {
					toast.success($i18n.t('OpenAI API settings updated'));
					await models.set(await getModels());
				}
			} catch (error) {
				toast.error(`${error}`);
			}
		}
	};

	const updateOllamaHandler = async () => {
		if (ENABLE_OLLAMA_API !== null) {
			OLLAMA_BASE_URLS = OLLAMA_BASE_URLS.map((url) => url.replace(/\/$/, ''));

			try {
				const res = await updateOllamaConfig(localStorage.token, {
					ENABLE_OLLAMA_API: ENABLE_OLLAMA_API,
					OLLAMA_BASE_URLS: OLLAMA_BASE_URLS,
					OLLAMA_API_CONFIGS: OLLAMA_API_CONFIGS
				});

				if (res) {
					toast.success($i18n.t('Ollama API settings updated'));
					await models.set(await getModels());
				}
			} catch (error) {
				toast.error(`${error}`);
			}
		}
	};

	const updateDirectConnectionsHandler = async () => {
		try {
			const res = await setDirectConnectionsConfig(localStorage.token, directConnectionsConfig);
			if (res) {
				toast.success($i18n.t('Direct Connections settings updated'));
				await models.set(await getModels());
			}
		} catch (error) {
			toast.error(`${error}`);
		}
	};

	const addOpenAIConnectionHandler = async (connection: {url: string, key: string, config: ApiConfig}) => {
		OPENAI_API_BASE_URLS = [...OPENAI_API_BASE_URLS, connection.url];
		OPENAI_API_KEYS = [...OPENAI_API_KEYS, connection.key];
		OPENAI_API_CONFIGS[OPENAI_API_BASE_URLS.length - 1] = connection.config;

		await updateOpenAIHandler();
	};

	const addOllamaConnectionHandler = async (connection: {url: string, key: string, config: ApiConfig}) => {
		OLLAMA_BASE_URLS = [...OLLAMA_BASE_URLS, connection.url];
		OLLAMA_API_CONFIGS[OLLAMA_BASE_URLS.length - 1] = {
			...connection.config,
			key: connection.key
		};

		await updateOllamaHandler();
	};

	onMount(async () => {
		if ($user?.role === 'admin') {
			try {
				let ollamaConfig: any = {};
				let openaiConfig: any = {};

				await Promise.all([
					(async () => {
						ollamaConfig = await getOllamaConfig(localStorage.token);
					})(),
					(async () => {
						openaiConfig = await getOpenAIConfig(localStorage.token);
					})(),
					(async () => {
						directConnectionsConfig = await getDirectConnectionsConfig(localStorage.token);
					})()
				]);

				ENABLE_OPENAI_API = openaiConfig.ENABLE_OPENAI_API;
				ENABLE_OLLAMA_API = ollamaConfig.ENABLE_OLLAMA_API;

				OPENAI_API_BASE_URLS = openaiConfig.OPENAI_API_BASE_URLS || [''];
				OPENAI_API_KEYS = openaiConfig.OPENAI_API_KEYS || [''];
				OPENAI_API_CONFIGS = openaiConfig.OPENAI_API_CONFIGS || {};

				OLLAMA_BASE_URLS = ollamaConfig.OLLAMA_BASE_URLS || [''];
				OLLAMA_API_CONFIGS = ollamaConfig.OLLAMA_API_CONFIGS || {};

				if (ENABLE_OPENAI_API) {
					for (const [idx, url] of OPENAI_API_BASE_URLS.entries()) {
						if (!OPENAI_API_CONFIGS[idx]) {
							OPENAI_API_CONFIGS[idx] = OPENAI_API_CONFIGS[url] || {};
						}
					}

					await Promise.all(
						OPENAI_API_BASE_URLS.map(async (url, idx) => {
							OPENAI_API_CONFIGS[idx] = OPENAI_API_CONFIGS[idx] || {};
							if (!(OPENAI_API_CONFIGS[idx]?.enable ?? true)) {
								return;
							}
							try {
								const res = await getOpenAIModels(localStorage.token, idx);
								if (res.pipelines) {
									pipelineUrls[url] = true;
								}
							} catch (error) {
								console.error('Error fetching OpenAI models:', error);
							}
						})
					);
				}

				if (ENABLE_OLLAMA_API) {
					for (const [idx, url] of OLLAMA_BASE_URLS.entries()) {
						if (!OLLAMA_API_CONFIGS[idx]) {
							OLLAMA_API_CONFIGS[idx] = OLLAMA_API_CONFIGS[url] || {};
						}
					}
				}
			} catch (error) {
				console.error('Error loading configs:', error);
				toast.error('Failed to load configuration');
			} finally {
				isLoading = false;
			}
		} else {
			isLoading = false;
		}
	});

	const submitHandler = async () => {
		await Promise.all([
			updateOpenAIHandler(),
			updateOllamaHandler(),
			updateDirectConnectionsHandler()
		]);
		dispatch('save');
	};
</script>

<AddConnectionModal
	bind:show={showAddOpenAIConnectionModal}
	onSubmit={addOpenAIConnectionHandler}
/>

<AddConnectionModal
	ollama
	bind:show={showAddOllamaConnectionModal}
	onSubmit={addOllamaConnectionHandler}
/>

<form class="flex flex-col h-full justify-between text-sm" on:submit|preventDefault={submitHandler}>
	<div class="overflow-y-auto scrollbar-hidden h-full">
		{#if isLoading}
			<div class="flex h-full justify-center">
				<div class="my-auto">
					<Spinner className="size-6" />
				</div>
			</div>
		{:else if $user?.role === 'admin'}
			<div class="my-2">
				<div class="mt-2 space-y-2 pr-1.5">
					<div class="flex justify-between items-center text-sm">
						<div class="font-medium">{$i18n.t('OpenAI API')}</div>

						<div class="flex items-center">
							<div class="">
								<Switch
									bind:state={ENABLE_OPENAI_API}
									on:change={updateOpenAIHandler}
								/>
							</div>
						</div>
					</div>

					{#if ENABLE_OPENAI_API}
						<hr class="border-gray-100 dark:border-gray-850" />

						<div class="">
							<div class="flex justify-between items-center">
								<div class="font-medium">{$i18n.t('Manage OpenAI API Connections')}</div>

								<Tooltip content={$i18n.t(`Add Connection`)}>
									<button
										class="px-1"
										on:click={() => {
											showAddOpenAIConnectionModal = true;
										}}
										type="button"
									>
										<Plus />
									</button>
								</Tooltip>
							</div>

							<div class="flex flex-col gap-1.5 mt-1.5">
								{#each OPENAI_API_BASE_URLS as url, idx (idx)}
									<OpenAIConnection
										pipeline={pipelineUrls[url] ? true : false}
										bind:url
										bind:key={OPENAI_API_KEYS[idx]}
										bind:config={OPENAI_API_CONFIGS[idx]}
										onSubmit={updateOpenAIHandler}
										onDelete={() => {
											OPENAI_API_BASE_URLS = OPENAI_API_BASE_URLS.filter(
												(_, urlIdx) => idx !== urlIdx
											);
											OPENAI_API_KEYS = OPENAI_API_KEYS.filter((_, keyIdx) => idx !== keyIdx);

											const newConfig = {};
											OPENAI_API_BASE_URLS.forEach((_, newIdx) => {
												newConfig[newIdx] = OPENAI_API_CONFIGS[newIdx < idx ? newIdx : newIdx + 1];
											});
											OPENAI_API_CONFIGS = newConfig;
											updateOpenAIHandler();
										}}
									/>
								{/each}
							</div>
						</div>
					{/if}
				</div>
			</div>

			<hr class="border-gray-100 dark:border-gray-850" />

			<div class="pr-1.5 my-2">
				<div class="flex justify-between items-center text-sm mb-2">
					<div class="font-medium">{$i18n.t('Ollama API')}</div>

					<div class="mt-1">
						<Switch
							bind:state={ENABLE_OLLAMA_API}
							on:change={updateOllamaHandler}
						/>
					</div>
				</div>

				{#if ENABLE_OLLAMA_API}
					<hr class="border-gray-100 dark:border-gray-850 my-2" />

					<div class="">
						<div class="flex justify-between items-center">
							<div class="font-medium">{$i18n.t('Manage Ollama API Connections')}</div>

							<Tooltip content={$i18n.t(`Add Connection`)}>
								<button
									class="px-1"
									on:click={() => {
										showAddOllamaConnectionModal = true;
									}}
									type="button"
								>
									<Plus />
								</button>
							</Tooltip>
						</div>

						<div class="flex w-full gap-1.5">
							<div class="flex-1 flex flex-col gap-1.5 mt-1.5">
								{#each OLLAMA_BASE_URLS as url, idx (idx)}
									<OllamaConnection
										bind:url
										bind:config={OLLAMA_API_CONFIGS[idx]}
										{idx}
										onSubmit={updateOllamaHandler}
										onDelete={() => {
											OLLAMA_BASE_URLS = OLLAMA_BASE_URLS.filter((_, urlIdx) => idx !== urlIdx);

											const newConfig = {};
											OLLAMA_BASE_URLS.forEach((_, newIdx) => {
												newConfig[newIdx] = OLLAMA_API_CONFIGS[newIdx < idx ? newIdx : newIdx + 1];
											});
											OLLAMA_API_CONFIGS = newConfig;
											updateOllamaHandler();
										}}
									/>
								{/each}
							</div>
						</div>

						<div class="mt-1 text-xs text-gray-400 dark:text-gray-500">
							{$i18n.t('Trouble accessing Ollama?')}
							<a
								class="text-gray-300 font-medium underline"
								href="https://github.com/open-webui/open-webui#troubleshooting"
								target="_blank"
							>
								{$i18n.t('Click here for help.')}
							</a>
						</div>
					</div>
				{/if}
			</div>

			<hr class="border-gray-100 dark:border-gray-850" />

			<div class="pr-1.5 my-2">
				<div class="flex justify-between items-center text-sm">
					<div class="font-medium">{$i18n.t('Direct Connections')}</div>

					<div class="flex items-center">
						<div class="">
							<Switch
								bind:state={directConnectionsEnabled}
								on:change={() => {
									if (directConnectionsConfig) {
										directConnectionsConfig.ENABLE_DIRECT_CONNECTIONS = directConnectionsEnabled;
									}
									updateDirectConnectionsHandler();
								}}
							/>
						</div>
					</div>
				</div>

				<div class="mt-1.5">
					<div class="text-xs text-gray-500">
						{$i18n.t(
							'Direct Connections allow users to connect to their own OpenAI compatible API endpoints.'
						)}
					</div>
				</div>
			</div>
		{:else}
			<div class="flex h-full justify-center items-center">
				<div class="text-gray-500">You don't have permission to access these settings</div>
			</div>
		{/if}
	</div>

	{#if $user?.role === 'admin'}
		<div class="flex justify-end pt-3 text-sm font-medium">
			<button
				class="px-3.5 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 transition rounded-full"
				type="submit"
			>
				{$i18n.t('Save')}
			</button>
		</div>
	{/if}
</form>
