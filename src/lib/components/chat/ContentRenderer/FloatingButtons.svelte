<script lang="ts">
    import { toast } from 'svelte-sonner';
    import DOMPurify from 'dompurify';
    import { marked } from 'marked';
    import { getContext, tick } from 'svelte';

    const i18n = getContext('i18n');
    import { chatCompletion } from '$lib/apis/openai';

    import LightBulb from '$lib/components/icons/LightBulb.svelte';
    import Markdown from '../Messages/Markdown.svelte';
    import Skeleton from '../Messages/Skeleton.svelte';

    export let id = '';
    export let model = null;
    export let messages = [];
    export let onAdd = () => {};

    let floatingInput = false;
    let selectedText = '';
    let floatingInputValue = '';
    let prompt = '';
    let responseContent = null;
    let responseDone = false;

    const autoScroll = async () => {
        const responseContainer = document.getElementById('response-container');
        if (
            responseContainer.scrollHeight - responseContainer.clientHeight <=
            responseContainer.scrollTop + 50
        ) {
            responseContainer.scrollTop = responseContainer.scrollHeight;
        }
    };

    const askHandler = async () => {
        if (!model) {
            toast.error('Model not selected');
            return;
        }
        prompt = [
            ...selectedText.split('\n').map((line) => `> ${line}`),
            '',
            floatingInputValue
        ].join('\n');
        floatingInputValue = '';

        responseContent = '';
        const [res, controller] = await chatCompletion(localStorage.token, {
            model: model,
            messages: [
                ...messages,
                {
                    role: 'user',
                    content: prompt
                }
            ].map((message) => ({
                role: message.role,
                content: message.content
            })),
            stream: true
        });

        if (res && res.ok) {
            const reader = res.body.getReader();
            const decoder = new TextDecoder();

            const processStream = async () => {
                while (true) {
                    const { done, value } = await reader.read();
                    if (done) break;

                    const chunk = decoder.decode(value, { stream: true });
                    const lines = chunk.split('\n').filter((line) => line.trim() !== '');

                    for (const line of lines) {
                        if (line.startsWith('')) {
                            if (line.includes('[DONE]')) {
                                responseDone = true;
                                await tick();
                                autoScroll();
                                continue;
                            } else {
                                try {
                                    const data = JSON.parse(line.slice(6));
                                    if (data.choices && data.choices[0]?.delta?.content) {
                                        responseContent += data.choices[0].delta.content;
                                        autoScroll();
                                    }
                                } catch (e) {
                                    console.error(e);
                                }
                            }
                        }
                    }
                }
            };

            await processStream();
        } else {
            toast.error('An error occurred while fetching the explanation');
        }
    };

    const explainHandler = async () => {
        if (!model) {
            toast.error('Model not selected');
            return;
        }
        const quotedText = selectedText
            .split('\n')
            .map((line) => `> ${line}`)
            .join('\n');
        prompt = `${quotedText}\n\nExplain`;

        responseContent = '';
        const [res, controller] = await chatCompletion(localStorage.token, {
            model: model,
            messages: [
                ...messages,
                {
                    role: 'user',
                    content: prompt
                }
            ].map((message) => ({
                role: message.role,
                content: message.content
            })),
            stream: true
        });

        if (res && res.ok) {
            const reader = res.body.getReader();
            const decoder = new TextDecoder();

            const processStream = async () => {
                while (true) {
                    const { done, value } = await reader.read();
                    if (done) break;

                    const chunk = decoder.decode(value, { stream: true });
                    const lines = chunk.split('\n').filter((line) => line.trim() !== '');

                    for (const line of lines) {
                        if (line.startsWith('')) {
                            if (line.includes('[DONE]')) {
                                responseDone = true;
                                await tick();
                                autoScroll();
                                continue;
                            } else {
                                try {
                                    const data = JSON.parse(line.slice(6));
                                    if (data.choices && data.choices[0]?.delta?.content) {
                                        responseContent += data.choices[0].delta.content;
                                        autoScroll();
                                    }
                                } catch (e) {
                                    console.error(e);
                                }
                            }
                        }
                    }
                }
            };

            await processStream();
        } else {
            toast.error('An error occurred while fetching the explanation');
        }
    };

    const addHandler = async () => {
        const messages = [
            {
                role: 'user',
                content: prompt
            },
            {
                role: 'assistant',
                content: responseContent
            }
        ];

        onAdd({
            modelId: model,
            parentId: id,
            messages: messages
        });
        closeHandler(); // Clear preview after adding
    };

    const closeHandler = () => {
        responseContent = null;
        responseDone = false;
        floatingInput = false;
        floatingInputValue = '';
    };
</script>

<div class="flex gap-4 w-full flex-wrap md:flex-nowrap">
    <!-- Left Side: Buttons & Input -->
    <div class="w-full md:w-1/2 lg:w-1/3">
        {#if responseContent === null}
            {#if !floatingInput}
                <div class="flex flex-row gap-2 p-2 rounded-lg dark:text-gray-100">
                    <!-- Updated Button with Icon in Square Box + Label -->
                    <button
                        class="flex items-center gap-2 px-3 py-1 hover:bg-gray-50 dark:hover:bg-gray-800 rounded-md"
                        on:click={() => {
                            selectedText = window.getSelection().toString();
                            explainHandler();
                        }}
                        title="Explain"
                    >
                        <!-- Unified Container for Icon + Label -->
                        <span class="flex items-center gap-2 rounded-md px-3 py-1.5 border border-gray-300 dark:border-gray-600">
                            <!-- Icon -->
                            <span class="flex items-center justify-center rounded-sm p-1 border-r border-gray-300 dark:border-gray-600">
                                <LightBulb class="size-4 text-gray-700 dark:text-gray-300" />
                            </span>
                            <!-- Label -->
                            <span class="text-sm font-medium">Explain</span>
                        </span>
                    </button>
                </div>
            {:else}
                <!-- Your existing floating input block -->
                <div class="flex items-center dark:text-gray-100 border border-gray-100 dark:border-gray-850 rounded-full p-1">
                    <input
                        type="text"
                        id="message-input"
                        class="ml-4 outline-none w-full flex-1 text-sm"
                        placeholder={$i18n.t('Ask a question')}
                        bind:value={floatingInputValue}
                        on:keydown={(e) => e.key === 'Enter' && askHandler()}
                    />
                    <button
                        class={`p-2 rounded-full ${
                            floatingInputValue !== ''
                                ? 'bg-black text-white hover:bg-gray-900 dark:bg-white dark:text-black dark:hover:bg-gray-100'
                                : 'bg-gray-200 text-white dark:bg-gray-700 dark:text-gray-900'
                        }`}
                        on:click={askHandler}
                    >
                        <svg
                            xmlns="http://www.w3.org/2000/svg"
                            viewBox="0 0 16 16"
                            fill="currentColor"
                            class="size-4"
                        >
                            <path
                                fill-rule="evenodd"
                                d="M8 14a.75.75 0 0 1-.75-.75V4.56L4.03 7.78a.75.75 0 0 1-1.06-1.06l4.5-4.5a.75.75 0 0 1 1.06 0l4.5 4.5a.75.75 0 0 1-1.06 1.06L8.75 4.56v8.69A.75.75 0 0 1 8 14Z"
                                clip-rule="evenodd"
                            />
                        </svg>
                    </button>
                </div>
            {/if}
        {/if}
    </div>


    <!-- Right Side: Explanation Preview -->
    <div class="w-full">
    {#if responseContent !== null}
        <div class="relative bg-white dark:bg-gray-850 rounded-lg border border-gray-200 dark:border-gray-700 shadow-sm">
            <!-- Cancel Button -->
            <button
                class="absolute top-2 right-2 p-1 text-gray-400 hover:text-gray-600 dark:text-gray-400 dark:hover:text-gray-200"
                on:click={closeHandler}
                title="Cancel"
            >
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-4 h-4">
                    <path d="M6.293 6.293a1 1 0 011.414 0L10 8.586l2.293-2.293a1 1 0 111.414 1.414L11.414 10l2.293 2.293a1 1 0 01-1.414 1.414L10 11.414l-2.293 2.293a1 1 0 01-1.414-1.414L8.586 10 6.293 7.707a1 1 0 010-1.414z" />
                </svg>
            </button>

            <!-- Content Area -->
            <div class="p-4">
                <!-- Prompt -->
                <div class="mb-3 text-sm font-medium text-gray-700 dark:text-gray-300">
                    <Markdown id={`${id}-prompt`} content={prompt} />
                </div>

                <!-- Response -->
                <div class="text-sm text-gray-600 dark:text-gray-400">
                    {#if responseContent.trim() === ''}
                        <div class="h-4 bg-gray-100 dark:bg-gray-700 rounded animate-pulse w-3/4"></div>
                    {:else}
                        <Markdown id={`${id}-response`} content={responseContent} />
                    {/if}
                </div>

                {#if responseDone}
                    <div class="flex justify-end mt-3">
                        <button
                            class="px-3 py-1.5 text-xs font-medium bg-gray-800 hover:bg-gray-700 text-white dark:bg-gray-200 dark:text-gray-800 dark:hover:bg-gray-300 rounded"
                            on:click={addHandler}
                        >
                            {$i18n.t('Add')}
                        </button>
                    </div>
                {/if}
            </div>
        </div>
    {/if}
</div>
</div>