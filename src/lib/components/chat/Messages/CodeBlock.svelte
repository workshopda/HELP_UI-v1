<!-- CodeBlock.svelte -->
<script lang="ts">
    import { getContext, onMount, onDestroy } from 'svelte';
    import { artifactCode, showArtifacts, showControls } from '$lib/stores';
    import mermaid from 'mermaid';
    import { v4 as uuidv4 } from 'uuid';

    const i18n = getContext('i18n');

    // Props
    export let id: string = '';
    export let lang: string = '';
    export let code: string = '';

    let mermaidHtml: string | null = null;
    let syncInterval: NodeJS.Timeout | null = null;
    let lastSyncedCode: string = '';
    let isBoxView: boolean = false;
    let copySuccess: boolean = false;


    // Check if code should be displayed in box view (5 lines or less)
    $: {
        const lineCount = code.split('\n').length;
        isBoxView = lineCount <= 5;
    }

    // Initialize Mermaid with better browser compatibility
    function initializeMermaid() {
        try {
            const isDark = document.documentElement.classList.contains('dark') || 
                          (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
            
            mermaid.initialize({
                startOnLoad: false,
                theme: isDark ? 'dark' : 'default',
                securityLevel: 'loose',
                flowchart: {
                    useMaxWidth: true,
                    htmlLabels: true
                }
            });
        } catch (error) {
            console.warn('Mermaid initialization fallback:', error);
        }
    }

    // Enhanced Mermaid rendering with error handling
    async function drawMermaidDiagram() {
        try {
            if (typeof mermaid !== 'undefined' && await mermaid.parse(code)) {
                const uniqueId = `mermaid-${uuidv4()}`;
                const { svg } = await mermaid.render(uniqueId, code);
                mermaidHtml = svg;
                triggerArtifacts(); // Trigger artifact after rendering Mermaid
            }
        } catch (error) {
            console.error('Mermaid rendering error:', error);
            // Fallback to regular code display
            mermaidHtml = null;
        }
    }
    // Copy code to clipboard
    async function copyCode(event: MouseEvent) {
        event.preventDefault();
        event.stopPropagation();
        
        try {
            await navigator.clipboard.writeText(code);
            copySuccess = true;
            setTimeout(() => {
                copySuccess = false;
            }, 2000);
        } catch (error) {
            console.error('Failed to copy code:', error);
            // Fallback for older browsers
            const textArea = document.createElement('textarea');
            textArea.value = code;
            document.body.appendChild(textArea);
            textArea.select();
            document.execCommand('copy');
            document.body.removeChild(textArea);
            copySuccess = true;
            setTimeout(() => {
                copySuccess = false;
            }, 2000);
        }
    }
    // Enhanced artifact triggering with browser compatibility
    function triggerArtifacts(event?: MouseEvent | KeyboardEvent) {
        if (event) {
            // Prevent default behavior
            if (event.preventDefault) event.preventDefault();
            if (event.stopPropagation) event.stopPropagation();
            
            // Handle keyboard events
            if (event.type === 'keydown') {
                const keyEvent = event as KeyboardEvent;
                if (keyEvent.key !== 'Enter' && keyEvent.key !== ' ') {
                    return;
                }
            }
        }

        const artifactId = id || Date.now().toString();

        try {
            // Update stores with enhanced error handling
            if (artifactCode && artifactCode.set) {
                artifactCode.set({
                    code,
                    lang,
                    id: artifactId
                });
            }
            
            if (showControls && showControls.set) {
                showControls.set(true);
            }
            
            if (showArtifacts && showArtifacts.set) {
                showArtifacts.set(true);
            }

            console.log("Triggered artifact:", { code, lang, id: artifactId });
            
            // Start real-time synchronization
            startRealTimeSync();
            
        } catch (error) {
            console.error('Error triggering artifacts:', error);
        }
    }

    // Real-time code synchronization
    function startRealTimeSync() {
        // Clear existing interval
        if (syncInterval) {
            clearInterval(syncInterval);
        }

        // Set up new synchronization interval
        syncInterval = setInterval(() => {
            if (code !== lastSyncedCode) {
                lastSyncedCode = code;
                
                try {
                    if (artifactCode && artifactCode.set) {
                        artifactCode.set({
                            code,
                            lang,
                            id: id || Date.now().toString()
                        });
                    }
                    
                    // Re-render Mermaid if needed
                    if (lang === 'mermaid') {
                        drawMermaidDiagram();
                    }
                    
                    console.log("Real-time sync updated:", { code: code.substring(0, 50) + '...' });
                } catch (error) {
                    console.error('Error in real-time sync:', error);
                }
            }
        }, 1000); // Sync every second
    }

    // Stop real-time synchronization
    function stopRealTimeSync() {
        if (syncInterval) {
            clearInterval(syncInterval);
            syncInterval = null;
        }
    }

    // Handle keyboard events for accessibility
    function handleKeydown(event: KeyboardEvent) {
        if (event.key === 'Enter' || event.key === ' ') {
            triggerArtifacts(event);
        }
    }

    // On mount hook with enhanced initialization
    onMount(() => {
        try {
            initializeMermaid();
            
            if (lang === 'mermaid') {
                drawMermaidDiagram();
            } else {
                triggerArtifacts(); // Auto-trigger artifact for non-Mermaid code blocks
            }
            
            // Initialize last synced code
            lastSyncedCode = code;
            
        } catch (error) {
            console.error('Error in onMount:', error);
        }
    });

    // Cleanup on destroy
    onDestroy(() => {
        stopRealTimeSync();
    });
</script>

<!-- Outer rectangular board -->
<div class="code-board" class:box-view={isBoxView}>
    <!-- Single code box -->
            <div 
        class="code-box" 
        class:box-view={isBoxView}
        on:click={triggerArtifacts}
        on:keydown={handleKeydown}
        role="button"
        tabindex="0"
        aria-label="Click to view code in artifact"
    >
        <div class="code-header" class:box-view={isBoxView}>
            <span class="language-label">{lang}</span>
            {#if isBoxView && lang !== 'mermaid'}
                <button 
                    class="copy-button"
                    on:click={copyCode}
                    aria-label="Copy code to clipboard"
                    title="Copy code"
                >
                    {#if copySuccess}
                        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                            <polyline points="20,6 9,17 4,12"></polyline>
                        </svg>
                    {:else}
                        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                            <rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect>
                            <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path>
                        </svg>
                    {/if}
                </button>
            {/if}
        </div>

        {#if lang === 'mermaid' && mermaidHtml}
            <div class="code-content mermaid" class:box-view={isBoxView} innerHTML={mermaidHtml} />
        {:else}
            <pre class="code-content" class:box-view={isBoxView}><code>{code}</code></pre>
        {/if}
    </div>
</div>

<style>
.code-board {
    border: 2px solid #d1d5db;
    border-radius: 12px;
    padding: 12px;
    background-color: #f9fafb;
    max-width: 100%;
    margin: 1rem 0;
    transition: all 0.3s ease-in-out;
}

.code-board.box-view {
    background: linear-gradient(135deg, #f9fafb 0%, #f3f4f6 100%);
    border: 3px solid #e5e7eb;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.dark .code-board {
    background-color: #111827;
    border-color: #374151;
}

.dark .code-board.box-view {
    background: linear-gradient(135deg, #111827 0%, #1f2937 100%);
    border-color: #4b5563;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.code-box {
    border-radius: 2px 2px 8px 8px;
    overflow: hidden;
    background-color: #f3f4f6;
    color: #1f2937;
    font-family: ui-monospace, SFMono-Regular, Consolas, 'Liberation Mono', Menlo, monospace;
    font-size: 12px;
    line-height: 1.5;
    cursor: pointer;
    transition: all 0.2s ease-in-out;
    margin-bottom: 10px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
    outline: none;
}

.code-box.box-view {
    border-radius: 12px;
    background: linear-gradient(145deg, #ffffff 0%, #f8fafc 100%);
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1), inset 0 1px 0 rgba(255, 255, 255, 0.6);
    border: 1px solid #e2e8f0;
    min-height: 200px;
    display: flex;
    flex-direction: column;
    cursor: default;
}

.code-box:not(.box-view):focus {
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.3);
}

.dark .code-box {
    background-color: #1f2937;
    color: #e5e7eb;
}

.dark .code-box.box-view {
    background: linear-gradient(145deg, #1e293b 0%, #334155 100%);
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.3), inset 0 1px 0 rgba(255, 255, 255, 0.1);
    border-color: #475569;
}

.code-header {
    padding: 2px 4px;
    background-color: #f3f4f6;
    font-weight: 500;
    font-size: 8px;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: #6b7280;
    border-bottom: 1px solid #e5e7eb;
    border-radius: 6px 6px 0 0;
    transform: rotate(-5deg);
    transform-origin: top left;
    box-shadow: 0 3px 6px rgba(0, 0, 0, 0.06);
    transition: transform 0.2s ease-in-out;
    width: 60px;
    height: 60px;
    display: flex;
    align-items: center;
    justify-content: center;
}

.code-header.box-view {
    transform: none;
    border-radius: 12px 12px 0 0;
    width: 100%;
    height: auto;
    padding: 12px 16px;
    font-size: 12px;
    font-weight: 600;
    background: linear-gradient(90deg, #f1f5f9 0%, #e2e8f0 100%);
    border-bottom: 2px solid #cbd5e1;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
    justify-content: space-between;
}

.dark .code-header {
    background-color: #2d323d;
    color: #cbd5e1;
    border-bottom: 1px solid #3b3f48;
}

.dark .code-header.box-view {
    background: linear-gradient(90deg, #2d3748 0%, #4a5568 100%);
    border-bottom-color: #4a5568;
    color: #e2e8f0;
}

.copy-button {
    background: transparent;
    border: none;
    color: #6b7280;
    cursor: pointer;
    padding: 4px;
    border-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease-in-out;
}

.copy-button:hover {
    background-color: #e5e7eb;
    color: #374151;
}

.copy-button:active {
    transform: scale(0.95);
}

.dark .copy-button {
    color: #9ca3af;
}

.dark .copy-button:hover {
    background-color: #4b5563;
    color: #e5e7eb;
}

.code-content {
    padding: 4px;
    white-space: pre-wrap;
    word-break: break-word;
    overflow: hidden;
    width: 60px;
    height: 60px;
    font-size: 9px;
    line-height: 1.2;
    font-family: 'Fira Code', ui-monospace, SFMono-Regular, Menlo, monospace;
    background-color: #f9fafb;
    border-radius: 0 0 6px 6px;
    color: #1f2937;
    border: 1px solid #e5e7eb;
    border-top: none;
    transform: rotate(-5deg);
    transform-origin: top left;
    box-shadow: 0 3px 6px rgba(0, 0, 0, 0.06);
    transition: transform 0.2s ease-in-out;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
}

.code-content.box-view {
    transform: none;
    border-radius: 0 0 12px 12px;
    width: 100%;
    height: auto;
    min-height: 150px;
    padding: 16px;
    font-size: 14px;
    line-height: 1.6;
    background: linear-gradient(145deg, #ffffff 0%, #f8fafc 100%);
    border: none;
    box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.05);
    text-align: left;
    align-items: flex-start;
    justify-content: flex-start;
    overflow-x: auto;
    overflow-y: auto;
    max-height: 400px;
}

.dark .code-content {
    background-color: #1f2937;
    color: #e5e7eb;
    border: 1px solid #374151;
    border-top: none;
}

.dark .code-content.box-view {
    background: linear-gradient(145deg, #1e293b 0%, #334155 100%);
    box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.2);
    color: #f1f5f9;
}

.code-box:not(.box-view):hover .code-header:not(.box-view),
.code-box:not(.box-view):hover .code-content:not(.box-view) {
    transform: rotate(-3deg);
}

.code-box.box-view:hover {
    transform: translateY(-2px);
    box-shadow: 0 12px 24px rgba(0, 0, 0, 0.15), inset 0 1px 0 rgba(255, 255, 255, 0.6);
}

.dark .code-box.box-view:hover {
    box-shadow: 0 12px 24px rgba(0, 0, 0, 0.4), inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

.mermaid svg {
    max-width: 100%;
    height: auto;
}

.code-content.box-view code {
    background: none;
    padding: 0;
    font-size: inherit;
    color: inherit;
}

/* Scrollbar styling for box view */
.code-content.box-view::-webkit-scrollbar {
    width: 8px;
    height: 8px;
}

.code-content.box-view::-webkit-scrollbar-track {
    background: #f1f5f9;
    border-radius: 4px;
}

.code-content.box-view::-webkit-scrollbar-thumb {
    background: #cbd5e1;
    border-radius: 4px;
}

.code-content.box-view::-webkit-scrollbar-thumb:hover {
    background: #94a3b8;
}

.dark .code-content.box-view::-webkit-scrollbar-track {
    background: #2d3748;
}

.dark .code-content.box-view::-webkit-scrollbar-thumb {
    background: #4a5568;
}

.dark .code-content.box-view::-webkit-scrollbar-thumb:hover {
    background: #718096;
}

/* Animation for smooth transitions */
@keyframes fadeIn {
    from { opacity: 0; transform: translateY(10px); }
    to { opacity: 1; transform: translateY(0); }
}

.code-box.box-view {
    animation: fadeIn 0.3s ease-out;
}

/* Responsive design */
@media (max-width: 768px) {
    .code-content.box-view {
        font-size: 12px;
        padding: 12px;
    }
    
    .code-header.box-view {
        padding: 8px 12px;
        font-size: 10px;
    }
}
</style>