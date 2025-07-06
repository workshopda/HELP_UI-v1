<script lang="ts">
    import { DropdownMenu } from 'bits-ui';
    import { flyAndScale } from '$lib/utils/transitions';
    import { getContext, onMount, tick } from 'svelte';
    import { config, user, tools as _tools, mobile } from '$lib/stores';
    import { createPicker } from '$lib/utils/google-drive-picker';
    import { getTools } from '$lib/apis/tools';
    import Dropdown from '$lib/components/common/Dropdown.svelte';
    import Tooltip from '$lib/components/common/Tooltip.svelte';
    import DocumentArrowUpSolid from '$lib/components/icons/DocumentArrowUpSolid.svelte';
    import Switch from '$lib/components/common/Switch.svelte';
    import GlobeAltSolid from '$lib/components/icons/GlobeAltSolid.svelte';
    import WrenchSolid from '$lib/components/icons/WrenchSolid.svelte';
    import CameraSolid from '$lib/components/icons/CameraSolid.svelte';
    import PhotoSolid from '$lib/components/icons/PhotoSolid.svelte';
    import CommandLineSolid from '$lib/components/icons/CommandLineSolid.svelte';

    const i18n = getContext('i18n');

    export let selectedToolIds: string[] = [];
    export let selectedModels: string[] = [];
    export let fileUploadCapableModels: string[] = [];
    export let screenCaptureHandler: Function;
    export let uploadFilesHandler: Function;
    export let inputFilesHandler: Function;
    export let uploadGoogleDriveHandler: Function;
    export let uploadOneDriveHandler: Function;
    export let onClose: Function;

    let tools = {};
    let show = false;
    let showAllTools = false;

    $: if (show) {
        init();
    }

    let fileUploadEnabled = true;
    $: fileUploadEnabled =
        fileUploadCapableModels.length === selectedModels.length &&
        ($user?.role === 'admin' || $user?.permissions?.chat?.file_upload);

    const init = async () => {
        if ($_tools === null) {
            await _tools.set(await getTools(localStorage.token));
        }
        tools = $_tools.reduce((a, tool, i, arr) => {
            a[tool.id] = {
                name: tool.name,
                description: tool.meta.description,
                enabled: selectedToolIds.includes(tool.id)
            };
            return a;
        }, {});
    };

    const detectMobile = () => {
        const userAgent = navigator.userAgent || navigator.vendor || window.opera;
        return /android|iphone|ipad|ipod|windows phone/i.test(userAgent);
    };

    function handleFileChange(event) {
        const inputFiles = Array.from(event.target?.files);
        if (inputFiles && inputFiles.length > 0) {
            inputFilesHandler(inputFiles);
        }
    }
</script>

<!-- Hidden file input used to open the camera on mobile -->
<input
    id="camera-input"
    type="file"
    accept="image/*"
    capture="environment"
    on:change={handleFileChange}
    style="display: none;"
/>

<Dropdown
    bind:show
    on:change={(e) => {
        if (e.detail === false) {
            onClose();
        }
    }}
>
    <Tooltip content={$i18n.t('More')}>
        <slot />
    </Tooltip>

    <div slot="content">
        <DropdownMenu.Content
            class="w-full max-w-[240px] rounded-lg border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 shadow-md z-50 p-2 transition-all"
            sideOffset={8}
            alignOffset={-8}
            side="top"
            align="start"
            transition={flyAndScale}
        >
            {#if Object.keys(tools).length > 0}
                <div class="max-h-36 overflow-y-auto pr-1 space-y-1">
                    {#each Object.keys(tools) as toolId (toolId)}
                        <button
                            class="flex w-full justify-between items-center px-3 py-2 text-sm font-medium text-gray-800 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg transition"
                            on:click={() => {
                                tools[toolId].enabled = !tools[toolId].enabled;
                            }}
                        >
                            <div class="flex gap-2 items-center truncate">
                                <WrenchSolid class="size-4 text-gray-500" />
                                <span>{tools[toolId].name}</span>
                            </div>
                            <Switch
                                state={tools[toolId].enabled}
                                on:change={async (e) => {
                                    const state = e.detail;
                                    await tick();
                                    if (state) {
                                        selectedToolIds = [...selectedToolIds, toolId];
                                    } else {
                                        selectedToolIds = selectedToolIds.filter((id) => id !== toolId);
                                    }
                                }}
                            />
                        </button>
                    {/each}
                </div>

                {#if Object.keys(tools).length > 3}
                    <hr class="border-gray-200 dark:border-gray-700 my-2" />
                    <button
                        class="w-full flex justify-center items-center text-sm text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-300 py-1"
                        on:click={() => {
                            showAllTools = !showAllTools;
                        }}
                    >
                        {showAllTools ? $i18n.t('Show Less') : $i18n.t('Show All')}
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            fill="none"
                            viewBox="0 0 24 24"
                            stroke-width="2"
                            stroke="currentColor"
                            class="size-4 ml-1 transition-transform duration-200 {showAllTools ? 'rotate-180' : ''}"
                        >
                            <path stroke-linecap="round" stroke-linejoin="round" d="m19.5 8.25-7.5 7.5-7.5-7.5" />
                        </svg>
                    </button>
                {/if}

                <hr class="border-gray-200 dark:border-gray-700 my-2" />
            {/if}

            <Tooltip content={fileUploadEnabled ? '' : $i18n.t('You do not have permission to upload files.')}>
                <DropdownMenu.Item
                    class="flex gap-2 items-center px-3 py-2 text-sm font-medium text-gray-800 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg cursor-pointer transition"
                    on:click={() => {
                        if (fileUploadEnabled) {
                            screenCaptureHandler();
                        }
                    }}
                >
                    <CameraSolid class="size-4 text-gray-500" />
                    <span>{$i18n.t('Capture')}</span>
                </DropdownMenu.Item>
            </Tooltip>

            <Tooltip content={fileUploadEnabled ? '' : $i18n.t('You do not have permission to upload files.')}>
                <DropdownMenu.Item
                    class="flex gap-2 items-center px-3 py-2 text-sm font-medium text-gray-800 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg cursor-pointer transition"
                    on:click={() => {
                        if (fileUploadEnabled) {
                            uploadFilesHandler();
                        }
                    }}
                >
                    <DocumentArrowUpSolid class="size-4 text-gray-500" />
                    <span>{$i18n.t('Upload Files')}</span>
                </DropdownMenu.Item>
            </Tooltip>

            {#if fileUploadEnabled && $config?.features?.enable_google_drive_integration}
                <DropdownMenu.Item
                    class="flex gap-2 items-center px-3 py-2 text-sm font-medium text-gray-800 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg cursor-pointer transition"
                    on:click={() => {
                        uploadGoogleDriveHandler();
                    }}
                >
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" class="size-4">
                        <path d="M12 0L24 12H0Z" fill="#4285F4" />
                        <path d="M12 24L0 12H24Z" fill="#34A853" />
                        <path d="M12 24V0" fill="#FBBC05" />
                    </svg>
                    <span>{$i18n.t('Google Drive')}</span>
                </DropdownMenu.Item>
            {/if}

            {#if fileUploadEnabled && $config?.features?.enable_onedrive_integration}
                <DropdownMenu.Sub>
                    <DropdownMenu.SubTrigger
                        class="flex gap-2 items-center px-3 py-2 text-sm font-medium text-gray-800 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg cursor-pointer transition"
                    >
                        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" class="size-4">
                            <path
                                d="M12 0L24 12H0Z"
                                fill="#0078D4"
                            />
                        </svg>
                        <span>{$i18n.t('Microsoft OneDrive')}</span>
                    </DropdownMenu.SubTrigger>
                    <DropdownMenu.SubContent
                        class="w-full rounded-lg border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800 shadow-md p-1"
                        side="right"
                        align="start"
                    >
                        <DropdownMenu.Item
                            class="flex gap-2 items-center px-3 py-2 text-sm font-medium text-gray-800 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg cursor-pointer transition"
                            on:click={() => {
                                uploadOneDriveHandler('personal');
                            }}
                        >
                            <span>{$i18n.t('Personal')}</span>
                        </DropdownMenu.Item>
                        <DropdownMenu.Item
                            class="flex gap-2 items-center px-3 py-2 text-sm font-medium text-gray-800 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg cursor-pointer transition"
                            on:click={() => {
                                uploadOneDriveHandler('organizations');
                            }}
                        >
                            <span>{$i18n.t('Work/School')}</span>
                        </DropdownMenu.Item>
                    </DropdownMenu.SubContent>
                </DropdownMenu.Sub>
            {/if}
        </DropdownMenu.Content>
    </div>
</Dropdown>