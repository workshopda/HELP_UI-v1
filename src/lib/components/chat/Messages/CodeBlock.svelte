<script lang="ts">
    import { getContext, onMount, onDestroy } from 'svelte';
    import { artifactCode, showArtifacts, showControls } from '$lib/stores';
    import mermaid from 'mermaid';
    import { v4 as uuidv4 } from 'uuid';

    const i18n = getContext('i18n');

    // Props
    export let lang: string = '';
    export let code: string = '';
    export const id: string = ''; // Marked as const since it's not used internally

    // State
    let mermaidHtml: string = '';
    let mermaidError: string | null = null;
    let mermaidInitialized: boolean = false;
    let syncInterval: NodeJS.Timeout | null = null;
    let refreshInterval: NodeJS.Timeout | null = null;
    let lastSyncedCode: string = '';
    let isBoxView: boolean = false;
    let copySuccess: boolean = false;
    let downloadSuccess: boolean = false;
    let viewArtifactSuccess: boolean = false;
    let mathJaxLoaded: boolean = false;
    let latexHtml: string = '';

    // Always display as box view
    $: {
        isBoxView = true;
        if (lang === 'mermaid') {
            drawMermaidDiagram();
        } else if (lang === 'latex') {
            renderLatex();
        }
    }

    // Initialize Mermaid
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
                },
                fontFamily: 'inherit'
            });
            mermaidInitialized = true;
        } catch (error) {
            console.warn('Mermaid initialization fallback:', error);
        }
    }

    // Load MathJax with proper configuration
    function loadMathJax() {
        if (mathJaxLoaded) return Promise.resolve();
        
        return new Promise((resolve) => {
            if (typeof window.MathJax !== 'undefined') {
                mathJaxLoaded = true;
                return resolve(true);
            }
            
            window.MathJax = {
                tex: {
                    inlineMath: [['\\(', '\\)']],
                    displayMath: [['\\[', '\\]']],
                    processEscapes: true,
                    packages: {'[+]': ['amsmath', 'amsfonts', 'amssymb']}
                },
                options: {
                    ignoreHtmlClass: 'tex2jax_ignore',
                    processHtmlClass: 'tex2jax_process'
                },
                startup: {
                    ready: () => {
                        window.MathJax.startup.defaultReady();
                        mathJaxLoaded = true;
                        resolve(true);
                    }
                }
            };
            
            const script = document.createElement('script');
            script.src = 'https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js';
            script.id = 'MathJax-script';
            script.async = true;
            document.head.appendChild(script);
        });
    }

    // Render LaTeX content
    async function renderLatex() {
        if (lang !== 'latex' || !code) return;
        
        try {
            await loadMathJax();
            
            // Wait for MathJax to be fully ready
            await new Promise<void>((resolve) => {
                const checkReady = () => {
                    if (window.MathJax?.typesetPromise) {
                        resolve();
                    } else {
                        setTimeout(checkReady, 50);
                    }
                };
                checkReady();
            });

            // Create a temporary div for processing
            const tempDiv = document.createElement('div');
            tempDiv.innerHTML = `\\[${code}\\]`;
            document.body.appendChild(tempDiv);
            
            // Process the LaTeX
            await window.MathJax.typesetPromise([tempDiv]);
            latexHtml = tempDiv.innerHTML;
            document.body.removeChild(tempDiv);
            
        } catch (error) {
            console.error('LaTeX rendering error:', error);
            latexHtml = code; // Fallback to raw code
        }
    }

    // Enhanced Mermaid rendering
    async function drawMermaidDiagram() {
        if (lang !== 'mermaid' || !code) return;
        
        try {
            // Reset states
            mermaidHtml = '';
            mermaidError = null;
            
            // Initialize Mermaid if not done yet
            if (!mermaidInitialized) {
                initializeMermaid();
            }

            // First validate the syntax
            await mermaid.parse(code);
            
            // Generate unique ID for the diagram
            const containerId = `mermaid-container-${uuidv4()}`;
            
            // Render the diagram
            const { svg } = await mermaid.render(containerId, code);
            mermaidHtml = svg;
            
        } catch (error) {
            console.error('Mermaid rendering error:', error);
            mermaidError = error.message || 'Invalid Mermaid syntax';
            
            // Fallback: Try to render anyway
            try {
                const containerId = `mermaid-fallback-${uuidv4()}`;
                const { svg } = await mermaid.render(containerId, code);
                mermaidHtml = svg;
                mermaidError = null;
            } catch (fallbackError) {
                console.error('Mermaid fallback rendering failed:', fallbackError);
            }
        }
    }

    // View artifact
    function viewArtifact(event: MouseEvent) {
        event.preventDefault();
        event.stopPropagation();
        triggerArtifacts();
        
        viewArtifactSuccess = true;
        setTimeout(() => viewArtifactSuccess = false, 2000);
    }

    // Download code as file
    function downloadCode(event: MouseEvent) {
        event.preventDefault();
        event.stopPropagation();
        
        try {
            const blob = new Blob([code], { type: 'text/plain' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `code-${lang}-${id || Date.now().toString()}.${getFileExtension()}`;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
            
            downloadSuccess = true;
            setTimeout(() => downloadSuccess = false, 2000);
        } catch (error) {
            console.error('Failed to download code:', error);
        }
    }

    function getFileExtension(): string {
        switch (lang) {
            case 'python': return 'py';
            case 'javascript': return 'js';
            case 'typescript': return 'ts';
            case 'html': return 'html';
            case 'css': return 'css';
            case 'json': return 'json';
            case 'latex': return 'tex';
            default: return 'txt';
        }
    }

    // Copy code to clipboard
    async function copyCode(event: MouseEvent) {
        event.preventDefault();
        event.stopPropagation();
        
        try {
            await navigator.clipboard.writeText(code);
            copySuccess = true;
            setTimeout(() => copySuccess = false, 2000);
        } catch (error) {
            console.error('Failed to copy code:', error);
            const textArea = document.createElement('textarea');
            textArea.value = code;
            document.body.appendChild(textArea);
            textArea.select();
            document.execCommand('copy');
            document.body.removeChild(textArea);
            copySuccess = true;
            setTimeout(() => copySuccess = false, 2000);
        }
    }

    // Setup auto-refresh
    function startAutoRefresh() {
        stopAutoRefresh();
        refreshInterval = setInterval(() => {
            if (lang === 'mermaid') {
                drawMermaidDiagram();
            } else if (lang === 'latex') {
                renderLatex();
            }
        }, 60000); // 60 seconds
    }

    // Stop auto-refresh
    function stopAutoRefresh() {
        if (refreshInterval) {
            clearInterval(refreshInterval);
            refreshInterval = null;
        }
    }

    // Trigger artifact manually or automatically
    function triggerArtifacts(event?: MouseEvent | KeyboardEvent) {
        if (event) {
            event.preventDefault?.();
            event.stopPropagation?.();
            
            if (event.type === 'keydown') {
                const keyEvent = event as KeyboardEvent;
                if (keyEvent.key !== 'Enter' && keyEvent.key !== ' ') return;
            }
        }

        const artifactId = id || Date.now().toString();

        try {
            artifactCode?.set({
                code: lang === 'mermaid' && mermaidHtml ? mermaidHtml : code,
                lang,
                id: artifactId
            });
            
            showControls?.set(true);
            showArtifacts?.set(true);
            
            startRealTimeSync();
        } catch (error) {
            console.error('Error triggering artifacts:', error);
        }
    }

    // Real-time code synchronization
    function startRealTimeSync() {
        stopRealTimeSync();
        syncInterval = setInterval(() => {
            if (code !== lastSyncedCode) {
                lastSyncedCode = code;
                artifactCode?.set({
                    code: lang === 'mermaid' && mermaidHtml ? mermaidHtml : code,
                    lang,
                    id: id || Date.now().toString()
                });
            }
        }, 1000);
    }

    // Stop real-time synchronization
    function stopRealTimeSync() {
        if (syncInterval) {
            clearInterval(syncInterval);
            syncInterval = null;
        }
    }

    // Lifecycle hooks
    onMount(() => {
        initializeMermaid();
        loadMathJax();
        if (lang === 'mermaid') drawMermaidDiagram();
        if (lang === 'latex') renderLatex();
        lastSyncedCode = code;
        startAutoRefresh();
    });

    onDestroy(() => {
        stopRealTimeSync();
        stopAutoRefresh();
    });
</script>

<!-- Template section -->
<div class="code-board" class:box-view={isBoxView}>
    <div class="code-box" class:box-view={isBoxView} role="button" tabindex="0" aria-label="Code block">
        <div class="code-header" class:box-view={isBoxView}>
            <span class="language-label">{lang}</span>
            {#if isBoxView && (lang !== 'mermaid' && lang !== 'latex')}
                <div class="button-group">
                    <button 
                        class="action-button view-artifact-button"
                        on:click={viewArtifact}
                        aria-label="View artifact"
                        title="View artifact"
                    >
                        {#if viewArtifactSuccess}
                            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                                <polyline points="20,6 9,17 4,12"></polyline>
                            </svg>
                        {:else}
                            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                                <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"></path>
                                <circle cx="12" cy="12" r="3"></circle>
                            </svg>
                        {/if}
                    </button>
                    <button 
                        class="action-button download-button"
                        on:click={downloadCode}
                        aria-label="Download code"
                        title="Download code"
                    >
                        {#if downloadSuccess}
                            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                                <polyline points="20,6 9,17 4,12"></polyline>
                            </svg>
                        {:else}
                            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                                <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
                                <polyline points="7 10 12 15 17 10"></polyline>
                                <line x1="12" y1="15" x2="12" y2="3"></line>
                            </svg>
                        {/if}
                    </button>
                    <button 
                        class="action-button copy-button"
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
                </div>
            {:else if isBoxView && (lang === 'mermaid' || lang === 'latex')}
                <div class="button-group">
                    <button 
                        class="action-button refresh-button"
                        on:click={lang === 'mermaid' ? drawMermaidDiagram : renderLatex}
                        aria-label="Refresh {lang === 'mermaid' ? 'Mermaid diagram' : 'LaTeX rendering'}"
                        title="Refresh"
                    >
                        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                            <path d="M23 4v6h-6"></path>
                            <path d="M1 20v-6h6"></path>
                            <path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"></path>
                        </svg>
                    </button>
                </div>
            {/if}
        </div>

        {#if lang === 'mermaid'}
            {#if mermaidHtml}
                <div class="code-content mermaid-diagram" class:box-view={isBoxView}>
                    {@html mermaidHtml}
                </div>
            {:else if mermaidError}
                <div class="code-content mermaid-error" class:box-view={isBoxView}>
                    <div>Diagram Error:</div>
                    <pre>{mermaidError}</pre>
                    <div class="original-code">Original Mermaid Code:</div>
                    <pre class="mermaid-code">{code}</pre>
                </div>
            {/if}
        {:else if lang === 'latex'}
            <div class="code-content latex-content" class:box-view={isBoxView}>
                {@html latexHtml || `\\[${code}\\]`}
            </div>
        {:else}
            <pre class="code-content" class:box-view={isBoxView}><code>{code}</code></pre>
        {/if}
    </div>
</div>

<style>
    .code-board {
        margin: 1rem 0;
        border-radius: 8px;
        overflow: hidden;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    }

    .code-box {
        background: var(--code-bg);
        border-radius: 8px;
        overflow: hidden;
    }

    .code-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 0.5rem 1rem;
        background: var(--code-header-bg);
        color: var(--code-header-text);
        font-family: var(--font-mono);
        font-size: 0.875rem;
    }

    .language-label {
        font-weight: bold;
        text-transform: uppercase;
    }

    .button-group {
        display: flex;
        gap: 0.5rem;
    }

    .action-button {
        background: none;
        border: none;
        color: inherit;
        cursor: pointer;
        padding: 0.25rem;
        opacity: 0.7;
        transition: opacity 0.2s;
    }

    .action-button:hover {
        opacity: 1;
    }

    .code-content {
        margin: 0;
        padding: 1rem;
        overflow: auto;
        max-height: 600px;
        background: var(--code-bg);
    }

    pre code {
        font-family: var(--font-mono);
        font-size: 0.875rem;
        line-height: 1.5;
        color: var(--code-text);
    }

    /* Mermaid specific styles */
    .mermaid-diagram {
        width: 100%;
        display: flex;
        justify-content: center;
        align-items: center;
        padding: 1rem;
        overflow: auto;
        text-align: center;
    }

    .mermaid-diagram svg {
        max-width: 100%;
        height: auto;
        margin: 0 auto;
    }

    /* LaTeX specific styles */
    .latex-content {
        text-align: center;
        padding: 1rem;
        min-height: 60px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }

    .latex-content .MJX-TEX {
        margin: 0.5rem 0;
    }

    .mermaid-error {
        padding: 1rem;
        color: var(--error-color);
        background: var(--error-bg);
        border-left: 3px solid var(--error-color);
    }

    .mermaid-error pre {
        white-space: pre-wrap;
        margin: 0.5rem 0;
        padding: 0.5rem;
        background: rgba(0,0,0,0.05);
        border-radius: 4px;
    }

    .original-code {
        margin-top: 1rem;
        font-weight: bold;
    }

    .mermaid-code {
        max-height: 200px;
        overflow: auto;
        background: var(--code-bg);
        padding: 0.5rem;
        border-radius: 4px;
    }
</style>