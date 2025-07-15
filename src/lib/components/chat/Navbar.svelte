<script lang="ts">
    import { getContext } from 'svelte';
    import { toast } from 'svelte-sonner';

    import {
        WEBUI_NAME,
        banners,
        chatId,
        config,
        mobile,
        settings,
        showArchivedChats,
        showControls,
        showSidebar,
        temporaryChatEnabled,
        user
    } from '$lib/stores';

    import { slide } from 'svelte/transition';
    import { page } from '$app/stores';

    import ShareChatModal from '../chat/ShareChatModal.svelte';
    import ModelSelector from '../chat/ModelSelector.svelte';
    import Tooltip from '../common/Tooltip.svelte';
    import Menu from '$lib/components/layout/Navbar/Menu.svelte';
    import UserMenu from '$lib/components/layout/Sidebar/UserMenu.svelte';
    import MenuLines from '../icons/MenuLines.svelte';
    import AdjustmentsHorizontal from '../icons/AdjustmentsHorizontal.svelte';

    import PencilSquare from '../icons/PencilSquare.svelte';
    import Banner from '../common/Banner.svelte';

    const i18n = getContext('i18n');

    export let initNewChat: Function;
    export let shareEnabled: boolean = false;

    export let chat;
    export let history;
    export let selectedModels;
    export let showModelSelector = true;

    let showShareChatModal = false;
    let showDownloadChatModal = false;
	let selectedChatId = null;

    function newChat() {
        initNewChat();
    }
</script>

<ShareChatModal bind:show={showShareChatModal} chatId={$chatId} />

<button
    id="new-chat-button"
    class="hidden"
    on:click={() => {
        initNewChat();
    }}
    aria-label="New Chat"
/>

<nav class="sticky top-0 z-30 w-full py-1 -mb-8 flex flex-col items-center drag-region">
    <div class="flex items-center w-full pl-1.5 pr-1">
        <div
            class="bg-linear-to-b via-50% from-white via-white to-transparent dark:from-gray-900 dark:via-gray-900 dark:to-transparent pointer-events-none absolute inset-0 -bottom-7 z-[-1]"
        ></div>

        <div class="flex max-w-full w-full mx-auto px-1 pt-0.5 bg-transparent">
            <div class="flex items-center w-full max-w-full">
                {#if !$showSidebar}
                    <div class="flex flex-col items-center space-y-2 mr-1 text-gray-600 dark:text-gray-400">
                        <!-- Sidebar Toggle -->
                        <button
                            type="button"
                            aria-label="Toggle menu"
                            class="relative bg-transparent hover:bg-gray-100 text-gray-800 dark:text-white dark:hover:bg-gray-800 transition rounded-full p-1.5 w-12 h-12 group flex items-center justify-center"
                            on:click={() => {
                                showSidebar.set(!$showSidebar);
                            }}
                        >
                            <!-- Default Icon -->
                            <svg
                                class="icon-default absolute inset-0 m-auto size-6 transition-all duration-300 ease-in-out text-gray-800 dark:text-white group-hover:opacity-0 group-hover:scale-75"
                                viewBox="0 0 20 20"
                                fill="currentColor"
                                xmlns="http://www.w3.org/2000/svg"
                            >
                                <path
                                    fill-rule="evenodd"
                                    clip-rule="evenodd"
                                    d="M2.5 3C1.67157 3 1 3.67157 1 4.5V15.5C1 16.3284 1.67157 17 2.5 17H17.5C18.3284 17 19 16.3284 19 15.5V4.5C19 3.67157 18.3284 3 17.5 3H2.5ZM2 4.5C2 4.22386 2.22386 4 2.5 4H6V16H2.5C2.22386 16 2 15.7761 2 15.5V4.5ZM7 16H17.5C17.7761 16 18 15.7761 18 15.5V4.5C18 4.22386 17.7761 4 17.5 4H7V16Z"
                                />
                            </svg>

                            <!-- Hover Icon -->
                            <svg
                                class="icon-hover absolute inset-0 m-auto size-6 transition-all duration-300 ease-in-out opacity-0 scale-50 group-hover:opacity-100 group-hover:scale-100 text-gray-800 dark:text-white"
                                viewBox="0 -960 960 960"
                                fill="currentColor"
                                xmlns="http://www.w3.org/2000/svg"
                            >
                                <path d="M160-240H80v-480h80v480Zm320 0L240-480l240-240 56 56-143 144h487v80H393l144 144-57 56Z" />
                            </svg>
                        </button>

                        <!-- New Chat Button -->
                        {#if !$mobile}
                        <Tooltip content={$i18n.t('New Chat')}>
                            <button
                                type="button"
                                aria-label="Add New Chat"
                                class="bg-transparent hover:bg-gray-100 dark:hover:bg-gray-800 transition rounded-lg p-1.5 w-10 h-10 flex items-center justify-center border border-gray-300 dark:border-gray-600"
                                on:click={() => {
                                    newChat();
                                }}
                            >
                                <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    class="w-5 h-5 text-gray-700 dark:text-gray-300"
                                    fill="none"
                                    viewBox="0 0 24 24"
                                    stroke-width="1.5"
                                    stroke="currentColor"
                                    role="img"
                                    aria-hidden="true"
                                >
                                    <path
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                        d="M12 4.5v15m-7.5-7.5h15"
                                    />
                                </svg>
                            </button>
                        </Tooltip>
                    {/if}

                    <!-- Notes Icon -->
                    {#if ($config?.features?.enable_notes ?? false) && ($user?.role === 'admin' || ($user?.permissions?.features?.notes ?? true))}
                        <Tooltip content={$i18n.t('Notes')}>
                            <a
                                class="bg-transparent hover:bg-gray-100 dark:hover:bg-gray-800 transition rounded-lg p-1.5 w-10 h-10 flex items-center justify-center border border-gray-300 dark:border-gray-600"
                                href="/notes"
                                on:click={() => {
                                    selectedChatId = null;
                                    chatId.set('');
                                    if ($mobile) {
                                        showSidebar.set(false);
                                    }
                                }}
                                draggable="false"
                            >
                                <svg
                                    class="w-5 h-5 text-gray-700 dark:text-gray-300"
                                    aria-hidden="true"
                                    xmlns="http://www.w3.org/2000/svg"
                                    width="24"
                                    height="24"
                                    fill="none"
                                    viewBox="0 0 24 24"
                                >
                                    <path
                                        stroke="currentColor"
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                        stroke-width="2"
                                        d="M10 3v4a1 1 0 0 1-1 1H5m4 8h6m-6-4h6m4-8v16a1 1 0 0 1-1 1H6a1 1 0 0 1-1-1V7.914a1 1 0 0 1 .293-.707l3.914-3.914A1 1 0 0 1 9.914 3H18a1 1 0 0 1 1 1Z"
                                    />
                                </svg>
                            </a>
                        </Tooltip>
                    {/if}
                    </div>
                {/if}

                <div
                    class="flex-1 overflow-hidden max-w-full py-0.5
                        {$showSidebar ? 'ml-1' : ''}
                    "
                >
                    {#if showModelSelector}
                        <ModelSelector bind:selectedModels showSetDefault={!shareEnabled} />
                    {/if}
                </div>

                <div class="self-start flex flex-none items-center text-gray-600 dark:text-gray-400">
                    {#if shareEnabled && chat && (chat.id || $temporaryChatEnabled)}
                        <Menu
                            {chat}
                            {shareEnabled}
                            shareHandler={() => {
                                showShareChatModal = !showShareChatModal;
                            }}
                            downloadHandler={() => {
                                showDownloadChatModal = !showDownloadChatModal;
                            }}
                        >
                            <button
                                class="flex cursor-pointer px-2 py-2 rounded-xl hover:bg-gray-50 dark:hover:bg-gray-850 transition"
                                id="chat-context-menu-button"
                            >
                                <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    fill="none"
                                    viewBox="0 0 24 24"
                                    stroke-width="1.5"
                                    stroke="currentColor"
                                    class="size-5"
                                >
                                    <path
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                        d="M6.75 12a.75.75 0 1 1-1.5 0 .75.75 0 0 1 1.5 0ZM12.75 12a.75.75 0 1 1-1.5 0 .75.75 0 0 1 1.5 0ZM18.75 12a.75.75 0 1 1-1.5 0 .75.75 0 0 1 1.5 0Z"
                                    />
                                </svg>
                            </button>
                        </Menu>
                    {/if}
                </div>
            </div>
        </div>
    </div>

    {#if $temporaryChatEnabled && $chatId === 'local'}
        <div class="w-full z-30 text-center">
            <div class="text-xs text-gray-500">{$i18n.t('Temporary Chat')}</div>
        </div>
    {/if}

    {#if !history.currentId && !$chatId && ($banners.length > 0 ||
        ($config?.license_metadata?.type ?? null) === 'trial' ||
        (($config?.license_metadata?.seats ?? null) !== null && $config?.user_count > $config?.license_metadata?.seats))}
        <div class="w-full z-30 mt-5">
            <div class="flex flex-col gap-1 w-full">
                {#if ($config?.license_metadata?.type ?? null) === 'trial'}
                    <Banner
                        banner={{
                            type: 'info',
                            title: 'Trial License',
                            content: $i18n.t(
                                'You are currently using a trial license. Please contact support to upgrade your license.'
                            )
                        }}
                    />
                {/if}

                {#if ($config?.license_metadata?.seats ?? null) !== null && $config?.user_count > $config?.license_metadata?.seats}
                    <Banner
                        banner={{
                            type: 'error',
                            title: 'License Error',
                            content: $i18n.t(
                                'Exceeded the number of seats in your license. Please contact support to increase the number of seats.'
                            )
                        }}
                    />
                {/if}

                {#each $banners.filter(b =>
                    b.dismissible
                        ? !JSON.parse(localStorage.getItem('dismissedBannerIds') ?? '[]').includes(b.id)
                        : true
                ) as banner}
                    <Banner
                        {banner}
                        on:dismiss={(e) => {
                            const bannerId = e.detail;
                            localStorage.setItem(
                                'dismissedBannerIds',
                                JSON.stringify([
                                    bannerId,
                                    ...JSON.parse(localStorage.getItem('dismissedBannerIds') ?? '[]')
                                ].filter(id => $banners.find(b => b.id === id)))
                            );
                        }}
                    />
                {/each}
            </div>
        </div>
    {/if}
</nav>