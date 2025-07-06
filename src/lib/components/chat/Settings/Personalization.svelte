<script lang="ts">
    import Switch from '$lib/components/common/Switch.svelte';
    import { config, models, settings, user } from '$lib/stores';
    import { createEventDispatcher, onMount, getContext, tick } from 'svelte';
    import { toast } from 'svelte-sonner';
    import ManageModal from './Personalization/ManageModal.svelte';
    import Tooltip from '$lib/components/common/Tooltip.svelte';
    const dispatch = createEventDispatcher();

    const i18n = getContext('i18n');

    export const saveSettings: Function = () => {};

    let showManageModal = false;

    // Addons
    let enableMemory = false;

    onMount(async () => {
        enableMemory = $settings?.memory ?? false;
    });
</script>

<ManageModal bind:show={showManageModal} />

<form
    id="tab-personalization"
    class="flex flex-col h-full justify-between space-y-3 text-sm"
    on:submit|preventDefault={() => {
        dispatch('save');
    }}
>
    <div class="py-1 overflow-y-scroll max-h-[28rem] lg:max-h-full">
        <!-- Memory section removed -->

        <!-- Optional: Show a message if needed -->
        <div class="text-xs text-gray-600 dark:text-gray-400 px-1">
            {$i18n.t(
                'Personalize your experience. More options will appear here as they become available.'
            )}
        </div>
    </div>

    <div class="flex justify-end text-sm font-medium">
        <button
            class="px-3.5 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 transition rounded-full"
            type="submit"
        >
            {$i18n.t('Save')}
        </button>
    </div>
</form>