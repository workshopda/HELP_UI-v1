<script lang="ts">
    import Fuse from 'fuse.js';
    import { getContext, createEventDispatcher } from 'svelte';
    import { settings, WEBUI_NAME } from '$lib/stores';
    import { WEBUI_VERSION } from '$lib/constants';

    const i18n = getContext('i18n');
    const dispatch = createEventDispatcher();

    export let suggestionPrompts = [];
    export const className = ''; // Changed to const since it's not modified
    export let inputValue = '';

    let sortedPrompts = [];
    let activeCategory: string | null = null;
    let showPrompts = false;

    // Default fallback prompts grouped by category
    const fallbackPrompts = {
        learn: [
            { content: "Explain photosynthesis like I'm 12.", category: 'learn' },
            { content: "Teach me the basics of Python programming.", category: 'learn' },
            { content: "Help me understand the Pythagorean theorem.", category: 'learn' },
            { content: "What are the main causes of World War I?", category: 'learn' },
            { content: "Give me a quiz on Indian history.", category: 'learn' }
        ],
        create: [
            { content: "Give me project ideas for a science fair.", category: 'create' },
            { content: "Generate a mind map for a startup business.", category: 'create' },
            { content: "Help me brainstorm a YouTube channel name.", category: 'create' },
            { content: "Create a logo concept for a school club.", category: 'create' },
            { content: "Design a coding challenge for beginners.", category: 'create' }
        ],
        write: [
            { content: "Write a short story about space exploration.", category: 'write' },
            { content: "Help me write a poem about friendship.", category: 'write' },
            { content: "Draft an introduction for my research paper.", category: 'write' },
            { content: "Fix the grammar in this paragraph.", category: 'write' },
            { content: "Convert my notes into an essay format.", category: 'write' }
        ],
        work: [
            { content: "Help me create a daily study and project schedule for my semester.", category: 'work' },
            { content: "Summarize this technical paper for my professor.", category: 'work' },
            { content: "What are some tips to stay focused during online lectures and coding sessions?", category: 'work' },
            { content: "Draft a professional email to request an extension on my assignment.", category: 'work' },
            { content: "Explain how to use Excel formulas to manage my college expenses.", category: 'work' }
        ]
    };

    const categories = [
        { id: 'work', name: 'Work', icon: 'work' },
        { id: 'learn', name: 'Learn', icon: 'learn' },
        { id: 'create', name: 'Create', icon: 'create' },
        { id: 'write', name: 'Write', icon: 'write' }
    ];

    const fuseOptions = {
        keys: ['content', 'title', 'category'],
        threshold: 0.5
    };

    let fuse: Fuse<any>;
    let filteredPrompts: any[] = [];

    // Helper function to get all prompts (external + fallback)
    function getAllPrompts() {
        const allPrompts = [];
        
        // Add external prompts if available
        if (suggestionPrompts && suggestionPrompts.length > 0) {
            allPrompts.push(...suggestionPrompts);
        }
        
        // Add fallback prompts for each category
        Object.keys(fallbackPrompts).forEach(category => {
            allPrompts.push(...fallbackPrompts[category]);
        });
        
        return allPrompts;
    }

    // Initialize sorted prompts and Fuse
    $: {
        const allPrompts = getAllPrompts();
        sortedPrompts = [...allPrompts].sort(() => Math.random() - 0.5);
        fuse = new Fuse(sortedPrompts, fuseOptions);
        getFilteredPrompts(inputValue);
    }

    // Helper function to check if arrays are the same
    function arraysEqual(a: any[], b: any[]): boolean {
        if (a.length !== b.length) return false;
        for (let i = 0; i < a.length; i++) {
            if ((a[i].id ?? a[i].content) !== (b[i].id ?? b[i].content)) {
                return false;
            }
        }
        return true;
    }

    const getFilteredPrompts = (inputValue: string) => {
        if (inputValue.length > 500) {
            filteredPrompts = [];
            return;
        }

        let promptsToFilter = sortedPrompts;

        // Filter by category if one is selected
        if (activeCategory) {
            promptsToFilter = sortedPrompts.filter((prompt) => {
                // Check if prompt has explicit category
                if (prompt.category) {
                    return prompt.category.toLowerCase() === activeCategory;
                }
                // For prompts without category, assume they belong to fallback
                return false;
            });
            
            // If no prompts found with category, use fallback prompts for that category
            if (promptsToFilter.length === 0 && fallbackPrompts[activeCategory]) {
                promptsToFilter = fallbackPrompts[activeCategory];
            }
        }

        const newFilteredPrompts =
            inputValue.trim() && fuse && promptsToFilter.length > 0
                ? fuse.search(inputValue.trim()).map((result) => result.item).filter(item => 
                    !activeCategory || (item.category && item.category.toLowerCase() === activeCategory)
                  )
                : promptsToFilter;

        if (!arraysEqual(filteredPrompts, newFilteredPrompts)) {
            filteredPrompts = newFilteredPrompts;
        }
    };

    const handleCategoryClick = (category: { id: string }) => {
        if (activeCategory === category.id) {
            activeCategory = null;
            showPrompts = false;
            inputValue = ''; // Clear input when deselected
        } else {
            activeCategory = category.id;
            showPrompts = true;
        }
        getFilteredPrompts(inputValue);
        dispatch('categorySelected', {
            category: category.id,
            prompts: filteredPrompts.map(p => p.content).join('\n')
        });
    };

    // Update filtered prompts when inputValue changes
    $: getFilteredPrompts(inputValue);
</script>

<!-- Responsive container -->
<div class="mb-4 w-full px-4 sm:px-0 sm:w-[90%] max-w-md -mt-4 mx-auto relative">
    <!-- Category Buttons - Responsive grid -->
    <div class="grid grid-cols-2 sm:grid-cols-4 gap-2 mb-4">
        {#each categories as category}
            <button
                on:click={() => handleCategoryClick(category)}
                class:active={activeCategory === category.id}
                class="flex flex-col items-center justify-center p-2 rounded-lg border transition-transform duration-200 transform hover:-translate-y-1 h-14 sm:h-12 {activeCategory === category.id ? 'border-opacity-50 font-semibold' : ''}"
            >
                {#if category.icon === 'work'}
                    <svg class="w-4 h-4 mb-1 sm:mb-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 13.255A23.931 23.931 0 0112 15c-3.183 0-6.22-.62-9-1.745M16 6V4a2 2 0 00-2-2h-4a2 2 0 00-2 2v2m4 6h.01M5 20h14a2 2 0 002-2V8a2 2 0 00-2-2H5a2 2 0 00-2 2v10a2 2 0 002 2z"></path>
                    </svg>
                {:else if category.icon === 'learn'}
                    <svg class="w-4 h-4 mb-1 sm:mb-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6.253v13m0-13C10.832 5.477 9.246 5 7.5 5S4.168 5.477 3 6.253v13C4.168 18.477 5.754 18 7.5 18s3.332.477 4.5 1.253m0-13C13.168 5.477 14.754 5 16.5 5c1.747 0 3.332.477 4.5 1.253v13C19.832 18.477 18.247 18 16.5 18c-1.746 0-3.332.477-4.5 1.253"></path>
                    </svg>
                {:else if category.icon === 'create'}
                    <svg class="w-4 h-4 mb-1 sm:mb-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.663 17h4.673M12 3v1m6.364 1.636l-.707.707M21 12h-1M4 12H3m3.343-5.657l-.707-.707m2.828 9.9a5 5 0 117.072 0l-.548.547A3.374 3.374 0 0014 18.469V19a2 2 0 11-4 0v-.531c0-.895-.356-1.754-.988-2.386l-.548-.547z"></path>
                    </svg>
                {:else if category.icon === 'write'}
                    <svg class="w-4 h-4 mb-1 sm:mb-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2-2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"></path>
                    </svg>
                {/if}
                <span class="text-xs sm:text-[10px] font-medium">{category.name}</span>
            </button>
        {/each}
    </div>

    <!-- Prompts Display - Horizontal sliding panel -->
    {#if showPrompts && activeCategory && filteredPrompts.length > 0}
        <div class="absolute top-full left-0 right-0 mt-2 z-10 animate-slide-down">
            <div class="bg-white dark:bg-gray-800 rounded-lg shadow-lg overflow-hidden border border-gray-200 dark:border-gray-700">
                <div class="flex overflow-x-auto py-3 px-4 scrollbar-thin scrollbar-thumb-gray-300 dark:scrollbar-thumb-gray-600 gap-3">
                    {#each filteredPrompts as prompt, idx (prompt.id || prompt.content)}
                        <button
                            class="flex-shrink-0 w-48 p-3 rounded-lg border border-gray-200 dark:border-gray-700 hover:bg-gray-100 dark:hover:bg-gray-700 transition-all duration-200 transform hover:scale-[1.02] bg-white dark:bg-gray-800"
                            style="animation-delay: {idx * 60}ms"
                            on:click={() => {
                                dispatch('select', prompt.content);
                                showPrompts = false;
                                activeCategory = null;
                            }}
                        >
                            <div class="text-left">
                                <div class="font-medium line-clamp-2 text-sm">
                                    {prompt.content}
                                </div>
                                <div class="text-xs opacity-70 mt-1">
                                    {$i18n.t('Prompt')}
                                </div>
                            </div>
                        </button>
                    {/each}
                </div>
            </div>
        </div>
    {/if}
</div>

<style>
    /* Animation for the horizontal prompts */
    @keyframes slide-down {
        0% {
            opacity: 0;
            transform: translateY(-10px);
        }
        100% {
            opacity: 1;
            transform: translateY(0);
        }
    }

    .animate-slide-down {
        animation: slide-down 0.2s ease-out forwards;
    }

    /* Animation for individual prompt cards */
    @keyframes fadeIn {
        0% {
            opacity: 0;
            transform: translateY(5px);
        }
        100% {
            opacity: 1;
            transform: translateY(0);
        }
    }

    .flex > button {
        opacity: 0;
        animation-name: fadeIn;
        animation-duration: 0.3s;
        animation-fill-mode: forwards;
        animation-timing-function: ease;
    }
</style>