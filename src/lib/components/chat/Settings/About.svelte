<script lang="ts">
	import { getVersionUpdates } from '$lib/apis';
	import { getOllamaVersion } from '$lib/apis/ollama';
	import { WEBUI_BUILD_HASH, WEBUI_VERSION } from '$lib/constants';
	import { WEBUI_NAME, config, showChangelog } from '$lib/stores';
	import { compareVersion } from '$lib/utils';
	import { onMount, getContext } from 'svelte';

	import Tooltip from '$lib/components/common/Tooltip.svelte';

	const i18n = getContext('i18n');

	let ollamaVersion = '';

	let updateAvailable = null;
	let version = {
		current: '',
		latest: ''
	};

	const checkForVersionUpdates = async () => {
		updateAvailable = null;
		version = await getVersionUpdates(localStorage.token).catch((error) => {
			return {
				current: WEBUI_VERSION,
				latest: WEBUI_VERSION
			};
		});

		console.log(version);

		updateAvailable = compareVersion(version.latest, version.current);
		console.log(updateAvailable);
	};

	onMount(async () => {
		ollamaVersion = await getOllamaVersion(localStorage.token).catch((error) => {
			return '';
		});

		checkForVersionUpdates();
	});
</script>

<div id="tab-about" class="flex flex-col h-full justify-between space-y-3 text-sm mb-6">
	<div class="space-y-3 overflow-y-scroll max-h-[28rem] lg:max-h-full">
		<div>
			<div class="mb-2.5 text-sm font-medium flex space-x-2 items-center">
				<div>
					<!-- App Name and Version Placeholder -->
					<span class="font-semibold">YourAppName</span>
					<span class="text-xs text-gray-400">(Version Info)</span>
				</div>
			</div>

			<div class="flex w-full justify-between items-center">
				<div class="flex flex-col text-xs text-gray-700 dark:text-gray-200">
					

					<button
						class="underline flex items-center space-x-1 text-xs text-gray-500 dark:text-gray-500"
						on:click={() => {
							showChangelog.set(true);
						}}>
						<div>See what's new</div>
					</button>
				</div>

				<button
					class="text-xs px-3 py-1.5 bg-gray-100 hover:bg-gray-200 dark:bg-gray-850 dark:hover:bg-gray-800 transition rounded-lg font-medium"
					on:click={() => {
						checkForVersionUpdates();
					}}>
					Check for updates
				</button>
			</div>
		</div>

		<!-- Optional Ollama or Dependency Version -->
		{#if ollamaVersion}
			<hr class="border-gray-100 dark:border-gray-850" />
			<div>
				<div class="mb-2.5 text-sm font-medium">Ollama Version</div>
				<div class="flex w-full">
					<div class="flex-1 text-xs text-gray-700 dark:text-gray-200">
						{ollamaVersion ?? 'N/A'}
					</div>
				</div>
			</div>
		{/if}

		<hr class="border-gray-100 dark:border-gray-850" />

		<!-- Custom License or Organization Info -->
		{#if $config?.license_metadata}
			<div class="mb-2 text-xs">
				<span class="capitalize">{$config?.license_metadata?.type}</span> license purchased by
				<span class="capitalize">{$config?.license_metadata?.organization_name}</span>
			</div>
		{/if}

		
		<!-- Attribution for any third-party assets -->
		<div class="mt-2 text-xs text-gray-400 dark:text-gray-500">
			Emoji graphics provided by
			<a href="https://github.com/jdecked/twemoji" target="_blank">Twemoji</a>, licensed under
			<a href="https://creativecommons.org/licenses/by/4.0/" target="_blank">CC-BY 4.0</a>.
		</div>

		<!-- License Section -->
		<div>
			<pre class="text-xs text-gray-400 dark:text-gray-500">
Copyright (c) {new Date().getFullYear()} Your Company Name
All rights reserved.

[Insert your custom license terms or use a template like MIT, BSD, etc.]
</pre>
		</div>

		
	</div>
</div>
