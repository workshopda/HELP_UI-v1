<script lang="ts">
    import { createEventDispatcher } from 'svelte';
    export let content: string = '';
    export let className: string = '';
    export let prompt: string = '$ ';
    
    const dispatch = createEventDispatcher();

    let input = '';
    let history: string[] = [];
    let historyIndex = 0;
    let terminalRef: HTMLDivElement;
    let inputRef: HTMLInputElement;
    let isScrolledToBottom = true;

    // Scroll to bottom when new content arrives
    $: {
        if (terminalRef && isScrolledToBottom) {
            terminalRef.scrollTop = terminalRef.scrollHeight;
        }
    }

    function handleKeyDown(e: KeyboardEvent) {
        // Handle up/down arrow for history
        if (e.key === 'ArrowUp') {
            e.preventDefault();
            if (historyIndex > 0) {
                historyIndex--;
                input = history[historyIndex];
            }
        } else if (e.key === 'ArrowDown') {
            e.preventDefault();
            if (historyIndex < history.length - 1) {
                historyIndex++;
                input = history[historyIndex];
            } else if (historyIndex === history.length - 1) {
                historyIndex = history.length;
                input = '';
            }
        } else if (e.key === 'Enter') {
            e.preventDefault();
            if (input.trim()) {
                history.push(input);
                historyIndex = history.length;
                dispatch('command', input);
                input = '';
            }
        }
    }

    function focusInput() {
        inputRef?.focus();
    }
</script>
<!-- Simple formatContent function to escape HTML -->
<div
    bind:this={terminalRef}
    class={`terminal ${className}`}
    on:click={focusInput}
    tabindex="0"
    role="region"
    aria-label="Terminal"
>
    {@html formatContent(content)}
    {#if prompt}
        <div class="flex items-center">
            <span class="text-green-400 mr-2">{prompt}</span>
            <input
                bind:this={inputRef}
                bind:value={input}
                on:keydown={handleKeyDown}
                class="flex-1 bg-transparent outline-none text-white"
            />
        </div>
    {/if}
</div>