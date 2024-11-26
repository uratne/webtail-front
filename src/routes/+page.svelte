<script lang="ts">
    import { onMount, onDestroy, tick } from 'svelte';
    import { fade } from 'svelte/transition';
    
    function getCurrentUTCTimestamp(): string {
        const now = new Date();
        return now.toISOString().replace(/\.\d{3}Z$/, '');
    }
  
    let lines: Array<{ content: string; timestamp: string; type: 'data' | 'system' }> = [];
    let autoScroll = true;
    let terminalDiv: HTMLDivElement;
    let eventSource: EventSource | null = null;
    let applications: Array<any> = [];
    let selectedApplication: any = null;
    let isLoading = false;

    async function fetchApplications() {
        try {
            isLoading = true;
            const response = await fetch('/api/applications');
            applications = await response.json();
        } catch (error) {
            console.error('Failed to fetch applications:', error);
            lines = [...lines, {
                content: 'Failed to fetch applications',
                timestamp: getCurrentUTCTimestamp(),
                type: 'system'
            }];
        } finally {
            isLoading = false;
        }
    }
  
    async function scrollToBottom() {
        if (autoScroll && terminalDiv) {
            await tick();
            terminalDiv.scrollTo({
                top: terminalDiv.scrollHeight,
                behavior: 'smooth'
            });
        }
    }

    async function handleMessage(event: MessageEvent) {
        const message = JSON.parse(event.data);

        if (message.type === 'Data') {
            if (message.replace_last_row && lines.length > 0) {
                lines = [...lines.slice(0, -1), {
                    content: message.row,
                    timestamp: message.timestamp,
                    type: 'data'
                }];
            } else {
                lines = [...lines, {
                    content: message.row,
                    timestamp: message.timestamp,
                    type: 'data'
                }];
            }
        } else if (message.type === 'System') {
            lines = [...lines, {
                content: message.message,
                timestamp: message.timestamp,
                type: 'system'
            }];
        }

        await scrollToBottom();
    }
  
    function handleScroll() {
        if (!terminalDiv) return;
        const isAtBottom = Math.abs(terminalDiv.scrollHeight - terminalDiv.clientHeight - terminalDiv.scrollTop) < 10;
        autoScroll = isAtBottom;
    }

    async function setupSSEConnection(application: any) {
        // Close existing connection if any
        if (eventSource) {
            eventSource.close();
        }

        // Clear terminal
        lines = [{
            content: `Connected to application: ${JSON.stringify(application)}`,
            timestamp: getCurrentUTCTimestamp(),
            type: 'system'
        }];

        // Setup new connection
        const encodedApp = encodeURIComponent(JSON.stringify(application));
        eventSource = new EventSource(`/api/sse?application=${encodedApp}`);
        
        eventSource.onmessage = handleMessage;
        eventSource.onerror = async (error) => {
            console.error('SSE Error:', error);
            lines = [...lines, {
                content: 'Connection error',
                timestamp: getCurrentUTCTimestamp(),
                type: 'system'
            }];
            await scrollToBottom();
        };
    }

    async function handleApplicationSelect(event: Event) {
        const select = event.target as HTMLSelectElement;
        if (select.value) {
            const application = JSON.parse(select.value);
            selectedApplication = application;
            await setupSSEConnection(application);
        }
    }
  
    onMount(() => {
        // Set document title
        document.title = 'f.efs';

        // Add initial message
        lines = [{
            content: 'Please select an application to begin',
            timestamp: getCurrentUTCTimestamp(),
            type: 'system'
        }];
    });
  
    onDestroy(() => {
        if (eventSource) {
            eventSource.close();
        }
    });
</script>

<div class="terminal">
    <div class="controls">
        <div class="select-container">
            <img src="/favicon.png" alt="Logo" class="logo" />
            <select
                on:mousedown={() => {
                    fetchApplications();
                }}
                on:change={handleApplicationSelect}
                class:loading={isLoading}
                value={selectedApplication ? JSON.stringify(selectedApplication) : ""}
            >
                {#if !selectedApplication}
                    <option value="">Select an application</option>
                {/if}
                {#each applications as app}
                    <option value={JSON.stringify(app)}>
                        {app.SinglePod ? app.SinglePod : `${app.MultiPod.application} - ${app.MultiPod.pod_name}`}
                    </option>
                {/each}
            </select>
            {#if isLoading}
                <span class="loading-indicator">Loading...</span>
            {/if}
        </div>
        <label class="auto-scroll">
            <input type="checkbox" bind:checked={autoScroll} />
            Auto-scroll
        </label>
        <button on:click={() => lines = []}>Clear Terminal</button>
    </div>
  
    <div class="content" bind:this={terminalDiv} on:scroll={handleScroll}>
        {#each lines as line (line.timestamp + line.content)}
            <div class="line" class:system={line.type === 'system'} transition:fade={{ duration: 150 }}>
                <span class="timestamp">{new Date(line.timestamp + 'z').toLocaleTimeString()}</span>
                <span class="line-content">{line.content}</span>
            </div>
        {/each}
    </div>
</div>
  
<style>
    /* Existing styles */
    :global(html), :global(body) {
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
    }

    :global(*) {
        box-sizing: border-box;
    }

    :global(#svelte) {
        width: 100%;
        height: 100%;
        position: absolute;
        top: 0;
        left: 0;
    }

    .terminal {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: #000;
        color: #fff;
        font-family: 'Consolas', 'Monaco', monospace;
        font-size: 14px;
        line-height: 1.4;
        display: flex;
        flex-direction: column;
    }

    .controls {
        background-color: #1a1a1a;
        padding: 8px;
        display: flex;
        gap: 1rem;
        border-bottom: 1px solid #333;
        width: 100%;
        align-items: center;
    }

    /* New styles for select dropdown */
    .select-container {
        position: relative;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .logo {
        width: 36px;
        height: 36px;
        object-fit: contain;
        margin-right: 2px;
    }

    select {
        padding: 4px 12px;
        background-color: #333;
        border: 1px solid #444;
        color: #888;
        border-radius: 2px;
        cursor: pointer;
        min-width: 200px;
    }

    select:hover {
        background-color: #444;
    }

    select.loading {
        opacity: 0.7;
        cursor: wait;
    }

    .loading-indicator {
        color: #888;
        font-size: 0.9em;
    }

    /* Existing styles */
    .auto-scroll {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        color: #888;
    }

    button {
        padding: 4px 12px;
        background-color: #333;
        border: 1px solid #444;
        color: #888;
        border-radius: 2px;
        cursor: pointer;
    }

    button:hover {
        background-color: #444;
    }

    .content {
        flex: 1;
        overflow-y: auto;
        padding: 8px;
        width: 100%;
    }

    .line {
        display: flex;
        gap: 1rem;
        padding: 2px 0;
        width: 100%;
    }

    .line.system {
        color: #ffd700;
    }

    .timestamp {
        color: #666;
        min-width: 100px;
        flex-shrink: 0;
    }

    .line-content {
        flex: 1;
        white-space: pre-wrap;
        word-break: break-all;
    }

    .content::-webkit-scrollbar {
        width: 10px;
    }

    .content::-webkit-scrollbar-track {
        background: #0a0a0a;
    }

    .content::-webkit-scrollbar-thumb {
        background-color: #333;
        border-radius: 5px;
        border: 2px solid #0a0a0a;
    }

    input[type="checkbox"] {
        accent-color: #444;
    }
</style>