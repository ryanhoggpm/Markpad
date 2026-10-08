<script lang="ts" module>
	/** Height of the bottom strip: the title bar's 36px, so a tab reads the same at either edge. */
	export const TAB_DOCK_STRIP_HEIGHT = 36;
</script>

<script lang="ts">
	import { tabManager } from '../stores/tabs.svelte.js';
	import { settings } from '../stores/settings.svelte.js';
	import TabList from './TabList.svelte';

	/**
	 * The open documents docked outside the title bar (#884): a column down
	 * the left or right side, or a strip along the bottom edge.
	 *
	 * The same `TabList` the title bar draws, so every tab action (context
	 * menu, middle-click close, drag to reorder, move to another window) is the
	 * strip's own code rather than a second copy of it. `MarkdownViewer` insets
	 * the document by the dock's size while it is shown.
	 */
	let {
		position,
		showHome = false,
		ontabclick,
		oncloseTab,
	} = $props<{
		position: 'left' | 'right' | 'bottom';
		showHome?: boolean;
		ontabclick?: () => void;
		oncloseTab?: (id: string) => void;
	}>();

	const isColumn = $derived(position !== 'bottom');
	let isResizing = $state(false);

	function startResize(e: PointerEvent) {
		e.preventDefault();
		const target = e.currentTarget as HTMLElement;
		target.setPointerCapture?.(e.pointerId);

		const startX = e.clientX;
		const startWidth = settings.tabColumnWidth;
		const side = position;
		isResizing = true;
		document.body.style.cursor = 'col-resize';
		document.body.style.userSelect = 'none';

		const onMove = (moveEvent: PointerEvent) => {
			const deltaX = moveEvent.clientX - startX;
			settings.setTabColumnWidth(startWidth + (side === 'left' ? deltaX : -deltaX));
		};

		const onUp = (upEvent: PointerEvent) => {
			window.removeEventListener('pointermove', onMove);
			window.removeEventListener('pointerup', onUp);
			window.removeEventListener('pointercancel', onUp);
			try {
				target.releasePointerCapture?.(upEvent.pointerId);
			} catch {
				// Pointer capture may already be gone after a cancel path.
			}
			document.body.style.cursor = '';
			document.body.style.userSelect = '';
			isResizing = false;
		};

		window.addEventListener('pointermove', onMove);
		window.addEventListener('pointerup', onUp);
		window.addEventListener('pointercancel', onUp);
	}
</script>

<aside
	class="tab-dock on-{position}"
	class:tagged={tabManager.windowTag !== null}
	class:resizing={isResizing}
	style:width={isColumn ? `${settings.tabColumnWidth}px` : null}
	style:height={isColumn ? null : `${TAB_DOCK_STRIP_HEIGHT}px`}
	style:--tag-color={tabManager.windowTag?.color}>
	<TabList
		orientation={isColumn ? 'vertical' : 'horizontal'}
		inTitleBar={false}
		onnewTab={() => tabManager.addNewTab()}
		{showHome}
		{ontabclick}
		{oncloseTab} />
	{#if isColumn}
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<div class="tab-dock-resizer" onpointerdown={startResize}></div>
	{/if}
</aside>

<style>
	.tab-dock {
		position: fixed;
		z-index: 900;
		display: flex;
		box-sizing: border-box;
		background: var(--color-canvas-default);
		font-family: var(--win-font, 'Segoe UI', sans-serif);
	}

	.tab-dock.on-left,
	.tab-dock.on-right {
		top: 36px;
		bottom: 0;
		flex-direction: column;
	}

	.tab-dock.on-left {
		left: 0;
		border-right: 1px solid var(--color-border-muted);
	}

	.tab-dock.on-right {
		right: 0;
		border-left: 1px solid var(--color-border-muted);
	}

	.tab-dock.on-bottom {
		left: 0;
		right: 0;
		bottom: 0;
		align-items: center;
		border-top: 1px solid var(--color-border-muted);
	}

	/* The window tag, drawn once along the dock's inner edge: the counterpart
	   of the line under the strip in TitleBar.svelte. */
	.tab-dock.tagged::after {
		content: '';
		position: absolute;
		background: var(--tag-color);
		pointer-events: none;
	}

	.tab-dock.tagged.on-left::after,
	.tab-dock.tagged.on-right::after {
		top: 0;
		bottom: 0;
		width: 2px;
	}

	.tab-dock.tagged.on-left::after {
		right: -1px;
	}

	.tab-dock.tagged.on-right::after {
		left: -1px;
	}

	.tab-dock.tagged.on-bottom::after {
		left: 0;
		right: 0;
		top: -1px;
		height: 2px;
		z-index: 30;
	}

	.tab-dock-resizer {
		position: absolute;
		top: 0;
		bottom: 0;
		width: 6px;
		cursor: col-resize;
		z-index: 1;
	}

	.on-left .tab-dock-resizer {
		right: -3px;
	}

	.on-right .tab-dock-resizer {
		left: -3px;
	}

	.tab-dock-resizer:hover,
	.tab-dock.resizing .tab-dock-resizer {
		background: color-mix(in srgb, var(--color-accent-fg) 35%, transparent);
	}
</style>
