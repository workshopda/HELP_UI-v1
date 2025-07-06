<script lang="ts">
    import { toast } from 'svelte-sonner';
    import { onMount, getContext, createEventDispatcher } from 'svelte';
    import mermaid from 'mermaid';
    import { v4 as uuidv4 } from 'uuid';
    import XMark from '../icons/XMark.svelte';
    import ArrowsPointingOut from '../icons/ArrowsPointingOut.svelte';
    import Tooltip from '../common/Tooltip.svelte';
    import SvgPanZoom from '../common/SVGPanZoom.svelte';
    import ArrowDownTray from '../icons/ArrowDownTray.svelte';
    import ChevronUp from '../icons/ChevronUp.svelte';
    import ChevronUpDown from '../icons/ChevronUpDown.svelte';
    import CodeEditor from '$lib/components/common/CodeEditor.svelte';
    import { artifactCode, chatId, settings, showArtifacts, showControls, config } from '$lib/stores';
    import { copyToClipboard, createMessagesList } from '$lib/utils';
    import PyodideWorker from '$lib/workers/pyodide.worker.js?worker';
    
    const i18n = getContext('i18n');
    const dispatch = createEventDispatcher();

    export let overlay = false;
    export let history;

    // Store subscriptions
    let currentConfig;
    const configUnsubscribe = config.subscribe(value => {
        currentConfig = value;
    });

    // Enhanced proxy configuration with all supported types
    let proxyConfig = {
        enabled: false,
        host: '',
        port: '',
        username: '',
        password: '',
        type: 'http' // 'http', 'https', 'socks5'
    };

    let messages = [];
    let contents: Array<{
        type: string;
        content: string;
        code: string;
        collapsed?: boolean;
        copied?: boolean;
        saved?: boolean;
        running?: boolean;
        installing?: boolean;
        output?: {
            stdout: string;
            stderr: string;
            result: string;
            files: Array<{ type: string; data: string }>;
            installedPackages?: string[];
            executionTime?: number;
        };
    }> = [];
    let selectedContentIdx = 0;
    let currentTab: 'artifact' | 'code' | 'output' = 'code';
    let currentCode = '';
    let mermaidHtml = null;
    let iframeElement: HTMLIFrameElement;
    let waitingForInput = false;
    let inputPrompt = '';
    let inputValue = '';
    let activeWorker = null;

    // Enhanced Python syntax detection with more patterns
    const checkPythonCode = (str) => {
        const pythonSyntax = [
            'def ',
            'class ',
            'if __name__',
            'else:',
            'elif ',
            'try:',
            'except:',
            'finally:',
            'yield ',
            'lambda ',
            'assert ',
            'nonlocal ',
            'del ',
            'True',
            'False',
            'None',
            ' and ',
            ' or ',
            ' not ',
            ' in ',
            ' is ',
            ' with ',
            'import ',
            'from ',
            'print(',
            'len(',
            'range(',
            'enumerate(',
            'zip(',
            'open(',
            'str(',
            'int(',
            'float(',
            'list(',
            'dict(',
            'set(',
            'tuple(',
            'input('
        ];

        return pythonSyntax.some(syntax => str.includes(syntax));
    };

    // Enhanced library detection with comprehensive patterns
    const detectRequiredLibraries = (code) => {
        const libraries = new Set();
        
        // Common library mappings
        const libraryMappings = {
            'numpy': ['numpy', 'np'],
            'pandas': ['pandas', 'pd'],
            'matplotlib': ['matplotlib', 'pyplot', 'plt'],
            'seaborn': ['seaborn', 'sns'],
            'scikit-learn': ['sklearn', 'scikit-learn'],
            'requests': ['requests'],
            'beautifulsoup4': ['bs4', 'BeautifulSoup'],
            'pillow': ['PIL', 'Image'],
            'opencv-python': ['cv2'],
            'tensorflow': ['tensorflow', 'tf'],
            'torch': ['torch', 'pytorch'],
            'plotly': ['plotly'],
            'bokeh': ['bokeh'],
            'scipy': ['scipy'],
            'sympy': ['sympy'],
            'networkx': ['networkx', 'nx'],
            'flask': ['flask', 'Flask'],
            'django': ['django'],
            'fastapi': ['fastapi'],
            'sqlalchemy': ['sqlalchemy'],
            'psycopg2': ['psycopg2'],
            'pymongo': ['pymongo'],
            'redis': ['redis'],
            'celery': ['celery'],
            'pydantic': ['pydantic'],
            'httpx': ['httpx'],
            'aiohttp': ['aiohttp'],
            'websockets': ['websockets'],
            'asyncio': ['asyncio'],
            'threading': ['threading'],
            'multiprocessing': ['multiprocessing'],
            'concurrent': ['concurrent'],
            'joblib': ['joblib'],
            'dask': ['dask'],
            'xgboost': ['xgboost'],
            'lightgbm': ['lightgbm'],
            'catboost': ['catboost'],
            'statsmodels': ['statsmodels'],
            'pytz': ['pytz'],
            'dateutil': ['dateutil'],
            'openpyxl': ['openpyxl'],
            'xlrd': ['xlrd'],
            'xlwt': ['xlwt'],
            'lxml': ['lxml'],
            'html5lib': ['html5lib'],
            'chardet': ['chardet'],
            'cryptography': ['cryptography'],
            'paramiko': ['paramiko'],
            'pycryptodome': ['Crypto'],
            'jsonschema': ['jsonschema'],
            'pyyaml': ['yaml'],
            'toml': ['toml'],
            'click': ['click'],
            'typer': ['typer'],
            'tqdm': ['tqdm'],
            'colorama': ['colorama'],
            'rich': ['rich'],
            'tabulate': ['tabulate']
        };

        // Extract import statements
        const importRegex = /(?:^|\n)\s*(?:import\s+([^\s,]+(?:\s*,\s*[^\s,]+)*)|from\s+([^\s,]+)\s+import)/gm;
        let match;
        
        while ((match = importRegex.exec(code)) !== null) {
            const importedModule = match[1] || match[2];
            if (importedModule) {
                const moduleName = importedModule.split('.')[0].trim();
                
                // Find corresponding package
                for (const [packageName, aliases] of Object.entries(libraryMappings)) {
                    if (aliases.includes(moduleName)) {
                        libraries.add(packageName);
                        break;
                    }
                }
                
                // If not found in mappings, use the module name directly
                if (![...Object.values(libraryMappings)].flat().includes(moduleName)) {
                    libraries.add(moduleName);
                }
            }
        }

        // Additional pattern-based detection
        const patterns = [
            { pattern: /plt\.|pyplot\.|matplotlib/, pkg: 'matplotlib' },
            { pattern: /np\.|numpy\./, pkg: 'numpy' },
            { pattern: /pd\.|pandas\./, pkg: 'pandas' },
            { pattern: /sns\.|seaborn\./, pkg: 'seaborn' },
            { pattern: /cv2\.|opencv/, pkg: 'opencv-python' },
            { pattern: /tf\.|tensorflow\./, pkg: 'tensorflow' },
            { pattern: /torch\.|pytorch/, pkg: 'torch' },
            { pattern: /requests\./, pkg: 'requests' },
            { pattern: /BeautifulSoup|bs4\./, pkg: 'beautifulsoup4' },
            { pattern: /PIL\.|Image\./, pkg: 'pillow' },
            { pattern: /sklearn\.|scikit/, pkg: 'scikit-learn' },
            { pattern: /plotly\./, pkg: 'plotly' },
            { pattern: /scipy\./, pkg: 'scipy' },
            { pattern: /sympy\./, pkg: 'sympy' },
            { pattern: /flask\.|Flask/, pkg: 'flask' },
            { pattern: /django\./, pkg: 'django' },
            { pattern: /fastapi\./, pkg: 'fastapi' }
        ];

        patterns.forEach(({ pattern, pkg }) => {
            if (pattern.test(code)) {
                libraries.add(pkg);
            }
        });

        return Array.from(libraries);
    };

    // Execute via Pyodide with enhanced features
    const executePythonViaPyodide = (code) => {
        return new Promise((resolve) => {
            const requiredLibraries = detectRequiredLibraries(code);
            console.log('Detected required libraries:', requiredLibraries);

            const worker = new PyodideWorker();
            activeWorker = worker;
            let resolved = false;
            let executionTimeout;

            // Handle input requests from worker
            worker.onmessage = (event) => {
                if (event.data.type === 'input_request') {
                    waitingForInput = true;
                    inputPrompt = event.data.prompt;
                    return;
                }

                clearTimeout(executionTimeout);
                if (!resolved) {
                    resolved = true;
                    const { stdout, stderr, result, installedPackages, executionTime } = event.data;
                    
                    if (installedPackages && installedPackages.length > 0) {
                        console.log('Installed packages:', installedPackages);
                    }
                    
                    let files = [];
                    [stdout, result].forEach((stream) => {
                        if (stream) {
                            stream.split('\n').forEach((line) => {
                                if (line.startsWith('data:image/')) {
                                    const mimeType = line.split(',')[0].split(':')[1].split(';')[0];
                                    files.push({ type: mimeType, data: line });
                                } else if (line.startsWith('data:application/json')) {
                                    files.push({ type: 'application/json', data: line });
                                } else if (line.startsWith('data:text/html')) {
                                    files.push({ type: 'text/html', data: line });
                                }
                            });
                        }
                    });

                    resolve({ 
                        stdout, 
                        stderr, 
                        result, 
                        files, 
                        installedPackages,
                        executionTime 
                    });
                    worker.terminate();
                    activeWorker = null;
                }
            };

            worker.onerror = (err) => {
                clearTimeout(executionTimeout);
                if (!resolved) {
                    resolved = true;
                    console.error('Worker error:', err);
                    resolve({ 
                        stdout: null, 
                        stderr: `Worker error: ${err.message}`, 
                        result: null, 
                        files: null 
                    });
                    worker.terminate();
                    activeWorker = null;
                }
            };

            // Set execution timeout (120 seconds)
            executionTimeout = setTimeout(() => {
                if (!resolved) {
                    resolved = true;
                    worker.terminate();
                    activeWorker = null;
                    resolve({
                        stdout: null,
                        stderr: 'Execution Time Limit Exceeded (120s)',
                        result: null,
                        files: null
                    });
                }
            }, 120000);

            worker.postMessage({ 
                id: uuidv4(), 
                code, 
                packages: requiredLibraries,
                proxyConfig: proxyConfig.enabled ? proxyConfig : null,
                timeout: 120000
            });
        });
    };

    // Unified execution function with better error handling
    const checkAndExecutePython = async (code, token) => {
        if (!checkPythonCode(code)) {
            console.warn('Input does not appear to be valid Python code.');
            toast.warning('Not valid Python code.');
            return {
                success: false,
                error: 'Not valid Python code'
            };
        }

        let result;
        const requiredLibraries = detectRequiredLibraries(code);
        
        if (requiredLibraries.length > 0) {
            toast.info(`Detected libraries: ${requiredLibraries.join(', ')}`);
        }

        if (currentConfig?.code?.engine === 'jupyter') {
            result = await executePythonViaJupyter(code, token);
        } else {
            result = await executePythonViaPyodide(code);
        }

        return {
            success: !result.stderr || result.stderr.trim() === '',
            ...result
        };
    };

    // Enhanced Python code execution with installation progress
    async function runPythonCode(index: number) {
        const requiredLibraries = detectRequiredLibraries(contents[index].code);
        
        if (requiredLibraries.length > 0) {
            contents[index].installing = true;
            toast.info(`Installing libraries: ${requiredLibraries.join(', ')}`);
        }
        
        contents[index].running = true;
        contents[index].output = null; // Clear previous output
        contents = [...contents];
        
        try {
            const result = await checkAndExecutePython(
                contents[index].code,
                $chatId
            );

            contents[index].output = {
                stdout: result.stdout || '',
                stderr: result.stderr || '',
                result: result.result || '',
                files: result.files || [],
                installedPackages: result.installedPackages || [],
                executionTime: result.executionTime
            };

            if (result.stderr && result.stderr.trim() !== '') {
                toast.error('Python execution failed');
            } else {
                if (result.installedPackages && result.installedPackages.length > 0) {
                    toast.success(`Python executed successfully. Installed: ${result.installedPackages.join(', ')}`);
                } else {
                    toast.success('Python executed successfully');
                }
            }
        } catch (error) {
            console.error('Error executing Python:', error);
            toast.error(`Error executing Python: ${error.message}`);
            contents[index].output = {
                stdout: '',
                stderr: error.message,
                result: '',
                files: []
            };
        } finally {
            contents[index].running = false;
            contents[index].installing = false;
            contents = [...contents];
        }
    }

    // Handle input submission from user
    function submitInput() {
        if (!inputValue.trim()) return;
        
        if (activeWorker) {
            activeWorker.postMessage({
                type: 'input_reply',
                value: inputValue
            });
        }
        
        waitingForInput = false;
        inputValue = '';
    }

    // Proxy configuration functions
    function updateProxyConfig(config) {
        proxyConfig = { ...proxyConfig, ...config };
        localStorage.setItem('pythonProxyConfig', JSON.stringify(proxyConfig));
    }

    function loadProxyConfig() {
        try {
            const stored = localStorage.getItem('pythonProxyConfig');
            if (stored) {
                proxyConfig = { ...proxyConfig, ...JSON.parse(stored) };
            }
        } catch (error) {
            console.error('Error loading proxy config:', error);
        }
    }

    function handleIncomingCode(newCode) {
        if (!newCode?.code) return;
        const existingIndex = contents.findIndex(c => c.code === newCode.code);
        if (existingIndex >= 0) {
            selectedContentIdx = existingIndex;
        } else {
            const type = newCode.lang === 'mermaid' ? 'mermaid' :
                newCode.code.includes('<svg') ? 'svg' :
                newCode.lang === 'python' ? 'python' :
                'iframe';
            contents = [...contents, {
                type,
                content: newCode.code,
                code: newCode.code,
                collapsed: false
            }];
            selectedContentIdx = contents.length - 1;
        }
        currentTab = 'code';
        showControls.set(true);
        showArtifacts.set(true);
    }

    $: if (history) {
        messages = createMessagesList(history, history.currentId);
        getContents();
    } else {
        messages = [];
        getContents();
    }

    function collapseCodeBlock(index: number) {
        contents[index].collapsed = !contents[index].collapsed;
        contents = [...contents];
    }

    function saveCode(index: number) {
        contents[index].saved = true;
        setTimeout(() => {
            contents[index].saved = false;
            contents = [...contents];
        }, 1000);
    }

    async function copyCode(index: number) {
        contents[index].copied = true;
        try {
            await copyToClipboard(contents[index].code);
            setTimeout(() => {
                contents[index].copied = false;
                contents = [...contents];
            }, 1000);
        } catch (err) {
            console.error('Copy failed:', err);
            contents[index].copied = false;
            contents = [...contents];
        }
    }

    async function drawMermaidDiagram(code: string) {
        try {
            if (await mermaid.parse(code)) {
                const id = `mermaid-${uuidv4()}`;
                const { svg } = await mermaid.render(id, code);
                mermaidHtml = svg;
            }
        } catch (error) {
            console.error('Error rendering Mermaid:', error);
        }
    }

    function getContents() {
        if (typeof window !== 'undefined') {
            requestIdleCallback(() => {
                _getContents();
            });
        } else {
            _getContents();
        }
    }

    function _getContents() {
        const newContents = [];
        messages.forEach((message) => {
            if (message?.role !== 'user' && message?.content) {
                const codeBlockContents = message.content.match(/```[\s\S]*?```/g) || [];
                let codeBlocks = [];

                codeBlockContents.forEach((block) => {
                    try {
                        const firstNewLine = block.indexOf('\n');
                        const lang = firstNewLine > 0 
                            ? block.slice(3, firstNewLine).trim().toLowerCase() 
                            : '';
                        const code = firstNewLine > 0 
                            ? block.slice(firstNewLine + 1, -3).trim()
                            : block.slice(3, -3).trim();
                        
                        if (code) {
                            codeBlocks.push({ lang, code });
                        }
                    } catch (e) {
                        console.error('Error processing code block:', e);
                    }
                });

                let htmlContent = '';
                let cssContent = '';
                let jsContent = '';
                const remainingBlocks = [];

                codeBlocks.forEach((block) => {
                    const { lang, code } = block;
                    
                    switch (lang) {
                        case 'html':
                            htmlContent += code + '\n';
                            break;
                        case 'css':
                            cssContent += code + '\n';
                            break;
                        case 'javascript':
                        case 'js':
                            jsContent += code + '\n';
                            break;
                        case 'mermaid':
                            newContents.push({
                                type: 'mermaid',
                                content: code,
                                code: code,
                                collapsed: false
                            });
                            break;
                        case 'python':
                        case 'py':
                            newContents.push({
                                type: 'python',
                                content: code,
                                code: code,
                                collapsed: false
                            });
                            break;
                        default:
                            remainingBlocks.push(block);
                    }
                });

                const inlineHtml = message.content.match(/<html>[\s\S]*?<\/html>/gi) || [];
                const inlineCss = message.content.match(/<style>[\s\S]*?<\/style>/gi) || [];
                const inlineJs = message.content.match(/<script>[\s\S]*?<\/script>/gi) || [];

                inlineHtml.forEach((block) => {
                    htmlContent += block.replace(/<\/?html>/gi, '') + '\n';
                });
                
                inlineCss.forEach((block) => {
                    cssContent += block.replace(/<\/?style>/gi, '') + '\n';
                });
                
                inlineJs.forEach((block) => {
                    jsContent += block.replace(/<\/?script>/gi, '') + '\n';
                });

                if (htmlContent || cssContent || jsContent) {
                    const fullCode = `<!-- HTML -->\n${htmlContent}\n<!-- CSS -->\n${cssContent}\n<!-- JavaScript -->\n${jsContent}`;
                    const renderedContent = `<!DOCTYPE html>
                            <html lang="en">
                            <head>
                                <meta charset="UTF-8">
                                <meta name="viewport" content="width=device-width, initial-scale=1.0">
                                <${''}style>
                                    ${cssContent}
                                </${''}style>
                            </head>
                            <body>
                                ${htmlContent}
                                <${''}script>
                                    ${jsContent}
                                </${''}script>
                            </body>
                            </html>`;
                    
                    newContents.push({
                        type: 'iframe',
                        content: renderedContent,
                        code: fullCode,
                        collapsed: false
                    });
                }

                remainingBlocks.forEach((block) => {
                    const { lang, code } = block;
                    
                    if (lang === 'svg' || (!lang && code.trim().startsWith('<svg'))) {
                        newContents.push({
                            type: 'svg',
                            content: code,
                            code: code,
                            collapsed: false
                        });
                    }
                    else if (code) {
                        newContents.push({
                            type: 'text',
                            content: code,
                            code: code,
                            collapsed: false
                        });
                    }
                });
            }
        });

        contents = newContents;

        if (contents.length === 0) {
            showControls.set(false);
            showArtifacts.set(false);
        } else {
            selectedContentIdx = contents.length - 1;
            currentCode = contents[selectedContentIdx]?.code || '';
        }
    }

    function navigateContent(direction: 'prev' | 'next') {
        selectedContentIdx =
            direction === 'prev'
                ? Math.max(selectedContentIdx - 1, 0)
                : Math.min(selectedContentIdx + 1, contents.length - 1);
        if (contents[selectedContentIdx]) {
            currentCode = contents[selectedContentIdx].code || '';
        }
    }

    function switchTab(tab: 'artifact' | 'code' | 'output') {
        currentTab = tab;
    }

    const iframeLoadHandler = () => {
        if (iframeElement?.contentWindow) {
            iframeElement.contentWindow.addEventListener(
                'click',
                function (e) {
                    const target = e.target.closest('a');
                    if (target && target.href) {
                        e.preventDefault();
                        const url = new URL(target.href, iframeElement.baseURI);
                        if (url.origin === window.location.origin) {
                            iframeElement.contentWindow.history.pushState(
                                null,
                                '',
                                url.pathname + url.search + url.hash
                            );
                        } else {
                            console.info('External navigation blocked:', url.href);
                        }
                    }
                },
                true
            );
            iframeElement.contentWindow.addEventListener('mouseenter', function (e) {
                e.preventDefault();
                iframeElement.contentWindow.addEventListener('dragstart', (event) => {
                    event.preventDefault();
                });
            });
        }
    };

    const showFullScreen = () => {
        if (iframeElement?.requestFullscreen) {
            iframeElement.requestFullscreen();
        } else if (iframeElement?.webkitRequestFullscreen) {
            iframeElement.webkitRequestFullscreen();
        } else if (iframeElement?.msRequestFullscreen) {
            iframeElement.msRequestFullscreen();
        }
    };

    const downloadArtifact = () => {
        const blob = new Blob([contents[selectedContentIdx].content], { type: 'text/html' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `artifact-${$chatId}-${selectedContentIdx}.html`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    };

    const downloadCode = () => {
        const blob = new Blob([currentCode], { type: 'text/plain' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `code-${$chatId}-${selectedContentIdx}.txt`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    };

    onMount(async () => {
        loadProxyConfig();
        
        const unsubscribe = artifactCode.subscribe((newCode) => {
            if (newCode?.code) {
                handleIncomingCode(newCode);
            }
        });

        const initMermaid = () => {
            if (document.documentElement.classList.contains('dark')) {
                mermaid.initialize({
                    startOnLoad: true,
                    theme: 'dark',
                    securityLevel: 'loose'
                });
            } else {
                mermaid.initialize({
                    startOnLoad: true,
                    theme: 'default',
                    securityLevel: 'loose'
                });
            }
        };

        let mermaidTimeout;
        window.addEventListener('theme-change', () => {
            clearTimeout(mermaidTimeout);
            mermaidTimeout = setTimeout(initMermaid, 300);
        });

        initMermaid();

        return () => {
            unsubscribe();
            configUnsubscribe();
            clearTimeout(mermaidTimeout);
            if (activeWorker) {
                activeWorker.terminate();
                activeWorker = null;
            }
        };
        
    });
</script>

{#if waitingForInput}
    <div class="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50">
        <div class="bg-white dark:bg-gray-800 rounded-lg p-6 max-w-md w-full shadow-xl">
            <h3 class="text-lg font-medium mb-4 dark:text-white">{inputPrompt || 'Python Input Request'}</h3>
            <input
                type="text"
                bind:value={inputValue}
                class="w-full px-3 py-2 border rounded dark:bg-gray-700 dark:border-gray-600 dark:text-white"
                on:keydown={(e) => e.key === 'Enter' && submitInput()}
                autofocus
            />
            <div class="flex justify-end mt-4 space-x-2">
                <button
                    on:click={() => {
                        waitingForInput = false;
                        inputValue = '';
                    }}
                    class="px-4 py-2 text-sm font-medium text-gray-700 dark:text-gray-200 bg-gray-100 dark:bg-gray-600 rounded hover:bg-gray-200 dark:hover:bg-gray-500"
                >
                    Cancel
                </button>
                <button
                    on:click={submitInput}
                    class="px-4 py-2 text-sm font-medium text-white bg-blue-600 rounded hover:bg-blue-700"
                >
                    Submit
                </button>
            </div>
        </div>
    </div>
{/if}

<div class="flex flex-col h-full w-full bg-white dark:bg-gray-900">
    {#if contents.length > 0}
        <div class="flex items-center justify-between p-2 border-b dark:border-gray-700">
            <div class="flex items-center space-x-2">
                <button
                    on:click={() => navigateContent('prev')}
                    disabled={selectedContentIdx <= 0}
                    class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-50 disabled:cursor-not-allowed"
                >
                    <ChevronUp class="w-5 h-5 transform rotate-90" />
                </button>
                <span class="text-sm font-medium dark:text-white">
                    {selectedContentIdx + 1} / {contents.length}
                </span>
                <button
                    on:click={() => navigateContent('next')}
                    disabled={selectedContentIdx >= contents.length - 1}
                    class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-50 disabled:cursor-not-allowed"
                >
                    <ChevronUp class="w-5 h-5 transform -rotate-90" />
                </button>
            </div>

            <div class="flex items-center space-x-2">
                {#if contents[selectedContentIdx].type === 'iframe'}
                    <Tooltip text="Fullscreen">
                        <button
                            on:click={showFullScreen}
                            class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700"
                        >
                            <ArrowsPointingOut class="w-5 h-5" />
                        </button>
                    </Tooltip>
                    <Tooltip text="Download HTML">
                        <button
                            on:click={downloadArtifact}
                            class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700"
                        >
                            <ArrowDownTray class="w-5 h-5" />
                        </button>
                    </Tooltip>
                {:else}
                    <Tooltip text="Download Code">
                        <button
                            on:click={downloadCode}
                            class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700"
                        >
                            <ArrowDownTray class="w-5 h-5" />
                        </button>
                    </Tooltip>
                {/if}
                <Tooltip text="Close">
                    <button
                        on:click={() => showArtifacts.set(false)}
                        class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700"
                    >
                        <XMark class="w-5 h-5" />
                    </button>
                </Tooltip>
            </div>
        </div>

        <div class="flex border-b dark:border-gray-700">
            <button
                class={`px-4 py-2 text-sm font-medium ${currentTab === 'artifact' ? 'text-blue-600 border-b-2 border-blue-600 dark:text-blue-400 dark:border-blue-400' : 'text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-300'}`}
                on:click={() => switchTab('artifact')}
            >
                Artifact
            </button>
            <button
                class={`px-4 py-2 text-sm font-medium ${currentTab === 'code' ? 'text-blue-600 border-b-2 border-blue-600 dark:text-blue-400 dark:border-blue-400' : 'text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-300'}`}
                on:click={() => switchTab('code')}
            >
                Code
            </button>
        </div>

        <div class="flex-1 overflow-auto">
            {#if currentTab === 'artifact'}
                {#if contents[selectedContentIdx].type === 'mermaid'}
                    <div class="p-4">
                        {#if mermaidHtml}
                            {@html mermaidHtml}
                        {:else}
                            <div class="flex items-center justify-center h-full">
                                <button
                                    on:click={() => drawMermaidDiagram(contents[selectedContentIdx].code)}
                                    class="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
                                >
                                    Render Diagram
                                </button>
                            </div>
                        {/if}
                    </div>
                {:else if contents[selectedContentIdx].type === 'svg'}
                    <div class="p-4 overflow-auto">
                        <SvgPanZoom svgContent={contents[selectedContentIdx].content} />
                    </div>
                {:else if contents[selectedContentIdx].type === 'iframe'}
                    <iframe
                        bind:this={iframeElement}
                        srcDoc={contents[selectedContentIdx].content}
                        class="w-full h-full border-0"
                        on:load={iframeLoadHandler}
                        sandbox="allow-scripts allow-same-origin"
                    />
                {:else}
                    <div class="p-4 overflow-auto">
                        <pre class="whitespace-pre-wrap break-words dark:text-white"><code>{contents[selectedContentIdx].content}</code></pre>
                    </div>
                {/if}
            {:else if currentTab === 'code'}
                <div class="relative h-full">
                    <div class="absolute top-2 right-2 z-10 flex space-x-2">
                        <Tooltip text={contents[selectedContentIdx].copied ? 'Copied!' : 'Copy Code'}>
                            <button
                                on:click={() => copyCode(selectedContentIdx)}
                                class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700"
                            >
                                {#if contents[selectedContentIdx].copied}
                                    <span class="text-green-500 text-sm">✓</span>
                                {:else}
                                    <span class="text-gray-500 dark:text-gray-400 text-sm">Copy</span>
                                {/if}
                            </button>
                        </Tooltip>
                        {#if contents[selectedContentIdx].type === 'python'}
                            <!--<Tooltip text="Run Code">
                                <button
                                    on:click={() => runPythonCode(selectedContentIdx)}
                                    disabled={contents[selectedContentIdx].running || contents[selectedContentIdx].installing}
                                    class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-50 disabled:cursor-not-allowed"
                                >
                                    {#if contents[selectedContentIdx].running}
                                        <span class="text-sm text-gray-500 dark:text-gray-400">Running...</span>
                                    {:else if contents[selectedContentIdx].installing}
                                        <span class="text-sm text-gray-500 dark:text-gray-400">Installing...</span>
                                    {:else}
                                        <span class="text-sm text-blue-600 dark:text-blue-400">Run</span>
                                    {/if}
                                </button>
                            </Tooltip>-->
                        {/if}
                    </div>
                    <CodeEditor
                        value={contents[selectedContentIdx].code}
                        readOnly={true}
                        language={contents[selectedContentIdx].type === 'python' ? 'python' : contents[selectedContentIdx].type === 'mermaid' ? 'mermaid' : 'html'}
                        class="h-full"
                    />
                </div>

                {#if contents[selectedContentIdx].type === 'python' && contents[selectedContentIdx].output}
                    <div class="border-t dark:border-gray-700 p-4 overflow-auto">
                        {#if contents[selectedContentIdx].output.installedPackages?.length > 0}
                            <div class="mb-4 p-3 bg-green-100 dark:bg-green-900 rounded">
                                <h4 class="font-medium text-green-800 dark:text-green-200">Installed Packages:</h4>
                                <p class="text-sm text-green-700 dark:text-green-300">
                                    {contents[selectedContentIdx].output.installedPackages.join(', ')}
                                </p>
                            </div>
                        {/if}

                        {#if contents[selectedContentIdx].output.executionTime}
                            <div class="mb-2 text-sm text-gray-500 dark:text-gray-400">
                                Execution time: {contents[selectedContentIdx].output.executionTime.toFixed(2)}ms
                            </div>
                        {/if}

                        {#if contents[selectedContentIdx].output.stdout}
                            <div class="mb-4">
                                <h4 class="font-medium dark:text-white">Output:</h4>
                                <pre class="whitespace-pre-wrap break-words bg-gray-100 dark:bg-gray-800 p-3 rounded text-sm"><code class="dark:text-gray-300">{contents[selectedContentIdx].output.stdout}</code></pre>
                            </div>
                        {/if}

                        {#if contents[selectedContentIdx].output.stderr}
                            <div class="mb-4">
                                <h4 class="font-medium text-red-600 dark:text-red-400">Errors:</h4>
                                <pre class="whitespace-pre-wrap break-words bg-red-50 dark:bg-red-900/20 p-3 rounded text-sm"><code class="text-red-600 dark:text-red-400">{contents[selectedContentIdx].output.stderr}</code></pre>
                            </div>
                        {/if}

                        {#if contents[selectedContentIdx].output.files?.length > 0}
                            <div class="mb-4">
                                <h4 class="font-medium dark:text-white">Generated Files:</h4>
                                <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mt-2">
                                    {#each contents[selectedContentIdx].output.files as file}
                                        {#if file.type.startsWith('image/')}
                                            <div class="border rounded overflow-hidden dark:border-gray-700">
                                                <img
                                                    src={file.data}
                                                    alt="Generated image"
                                                    class="w-full h-auto"
                                                />
                                            </div>
                                        {:else if file.type === 'application/json'}
                                            <div class="border rounded p-3 dark:border-gray-700">
                                                <pre class="text-xs overflow-auto max-h-60 dark:text-gray-300">{JSON.stringify(JSON.parse(atob(file.data.split(',')[1])), null, 2)}</pre>
                                            </div>
                                        {:else if file.type === 'text/html'}
                                            <div class="border rounded overflow-hidden dark:border-gray-700">
                                                <iframe
                                                    srcDoc={atob(file.data.split(',')[1])}
                                                    class="w-full h-64 border-0"
                                                    sandbox="allow-scripts allow-same-origin"
                                                />
                                            </div>
                                        {/if}
                                    {/each}
                                </div>
                            </div>
                        {/if}
                    </div>
                {/if}
            {/if}
        </div>
    {:else}
        <div class="flex items-center justify-center h-full">
            <p class="text-gray-500 dark:text-gray-400">No artifacts to display</p>
        </div>
    {/if}
</div>