<script lang="ts">
	/**
	 * This is a minimal example of sigma. You can use it as a base to write new
	 * examples, or reproducible test cases for new issues, for instance.
	 */

	import Graph from 'graphology';
	import Sigma from 'sigma';
	import dijkstra from 'graphology-shortest-path/dijkstra';
	import type { EdgeDisplayData, PlainObject } from 'sigma/types';

	let bgnd: HTMLCanvasElement;
	let sum = 0;

	interface GraphData {
		directed: boolean;
		multigraph: boolean;
		nodes: [
			{
				id: number;
				is_black: boolean;
				x: number;
				y: number;
			}
		];
		links: [
			{
				source: number;
				target: number;
				weight: number;
				speed: number;
				color: string;
			}
		];
		width: number;
		height: number;
	}

	interface State {
		path: string[];
	}

	let renderer = null as Sigma | null;
	let from: string | null = null;
	let to: string | null = null;
	let graph: Graph;
	const state: State = { path: [] };
	let container: HTMLElement | null = null;
	const img = new Image();
	let graphWidth = 20;
	let graphHeight = 20;
	let nodeMinX = Infinity;
	let nodeMinY = Infinity;

	function init(node: HTMLElement) {
		container = document.getElementById('sigma-container');
		if (!container) throw 'No container found';
		console.log(container);

		graph = new Graph();

		graph.addNode('John', { x: 0, y: 10, size: 5, label: 'John', color: 'blue' });
		graph.addNode('Mary', { x: -10, y: 0, size: 3, label: 'Mary', color: 'red' });

		graph.addEdge('John', 'Mary');

		// eslint-disable-next-line @typescript-eslint/no-unused-vars
		renderer = new Sigma(graph, container, {
			maxCameraRatio: 128,
			minCameraRatio: 0.01,
			renderEdgeLabels: true,
			defaultNodeColor: '#00000022',
			zoomToSizeRatioFunction: (x) => x,
			itemSizesReference: 'positions'
		});
		renderer.once('afterRender', () => {
			resizeBackgroundCanvas();
		});

		renderer.on('resize', () => {
			resizeBackgroundCanvas();
		});

		renderer.getCamera().on('updated', () => {
			refreshBackground();
		});
		img.addEventListener('load', () => {
			refreshBackground();
		});

		renderer.on('clickNode', (event) => {
			console.log(event);
			if (from === null) {
				from = event.node;
			} else if (to === null) {
				to = event.node;

				console.log(graph.nodes());
				const path = dijkstra.bidirectional(graph, from, to);

				if (path === null) {
					alert('No path found');
				} else {
					console.log(path);

					state.path = path;
					sum = 0;
					renderer?.refresh();
				}

				from = to = null;
			}
		});

		renderer.setSetting('edgeReducer', (edge, data) => {
			const res: Partial<EdgeDisplayData> = { ...data };
			// console.log(data, edge);
			const source = graph.source(edge);
			const target = graph.target(edge);
			let index = state.path.indexOf(source);
			if (index !== -1) {
				if (
					(state.path[index + 1] && state.path[index + 1] === target) ||
					(state.path[index - 1] && state.path[index - 1] === target)
				) {
					res.size = 5;
					sum += data.weight;
				}
			}

			return res;
		});
	}

	function resizeBackgroundCanvas() {
		if (!renderer) return;
		const nodeCanvas: HTMLCanvasElement = renderer.getCanvases()['nodes'];
		bgnd.width = nodeCanvas.width;
		bgnd.height = nodeCanvas.height;
		bgnd.style.width = nodeCanvas.style.width;
		bgnd.style.height = nodeCanvas.style.height;
		console.log(nodeCanvas);

		refreshBackground();
	}

	function refreshBackground() {
		// https://codesandbox.io/s/sigma-with-leaflet-lepn9?file=/index.ts
		console.log('updated cam');

		const ctx = bgnd.getContext('2d');
		if (!ctx || !renderer) return;
		ctx.clearRect(0, 0, bgnd.width, bgnd.height);
		ctx.fillStyle = 'white';

		// const viewportDim = renderer.getGraphDimensions();
		const topLeft = renderer.graphToViewport({ x: 0, y: 0 });
		const bottomRight = renderer.graphToViewport({ x: graphWidth, y: graphHeight });

		const pxRatio = window.devicePixelRatio;
		const rect = {
			x: Math.min(topLeft.x, bottomRight.x) * pxRatio,
			y: Math.min(topLeft.y, bottomRight.y) * pxRatio,
			width: Math.abs(topLeft.x - bottomRight.x) * pxRatio,
			height: Math.abs(topLeft.y - bottomRight.y) * pxRatio
		};
		rect.y += rect.height;
		// const topLeftNode = renderer.graphToViewport({ x: nodeMinX, y: nodeMinY });
		// rect.x += (topLeftNode.x - topLeft.x) * pxRatio;
		// rect.y += (topLeftNode.y - topLeft.y) * pxRatio;

		ctx.fillRect(rect.x, rect.y, rect.width, rect.height);
		ctx.imageSmoothingEnabled = false;
		ctx.globalAlpha = 0.35;
		ctx.drawImage(img, 0, 0, img.width, img.height, rect.x, rect.y, rect.width, rect.height);
		ctx.globalAlpha = 1;
	}

	/**
	 * Load a graph from a json file
	 * @param event
	 */
	function loadGraph(event: Event) {
		const fileInput = event.target as HTMLInputElement;
		const file = fileInput.files?.[0];
		if (!file) return;

		const reader = new FileReader();
		reader.onload = (event) => {
			const graph = JSON.parse(event.target?.result as string) as GraphData;
			console.log(graph);
			buildGraph(graph);
		};
		reader.readAsText(file);
	}

	/**
	 * Build the graph from the json data using graphology and display it using sigma
	 * @param graphJSON
	 */
	function buildGraph(graphJSON: GraphData) {
		nodeMinX = Infinity;
		nodeMinY = Infinity;
		graphWidth = graphJSON.width;
		graphHeight = graphJSON.height;
		graph = new Graph({
			multi: graphJSON.multigraph,
			type: graphJSON.directed ? 'directed' : 'undirected'
		});

		for (const node of graphJSON.nodes) {
			const x = node.x;
			const y = node.y * -1;
			nodeMinX = Math.min(nodeMinX, x);
			nodeMinY = Math.min(nodeMinY, y);
			graph.addNode(node.id, {
				x: x,
				y: y,
				label: node.id.toString() + ` (${x}, ${y})`,
				size: 3,
			});
		}

		for (const link of graphJSON.links) {
			graph.addEdge(link.source, link.target, {
				weight: link.weight,
				color: link.color + "aa",
				label: link.weight.toString(),
				size: 2,
			});
		}

		renderer?.setGraph(graph);
	}

	/**
	 * Define the background of the graph
	 * @param event
	 */
	function loadBackground(event: Event) {
		const fileInput = event.target as HTMLInputElement;
		const file = fileInput.files?.[0];
		if (!file) return;

		const reader = new FileReader();
		reader.onload = (event) => {
			if (!container) return;
			const url = event.target?.result;
			img.src = url as string;
		};
		reader.readAsDataURL(file);
	}
</script>

<div id="root">
	<input type="file" name="file" id="file-input" accept=".json" on:change={loadGraph} />
	<input
		type="file"
		name="file"
		id="bgnd-input"
		accept=".jpeg, .jpg, .png"
		on:change={loadBackground}
	/>
	<span>Sum: <span>{sum}</span></span>
	<div id="sigma-container" use:init>
		<canvas bind:this={bgnd} id="background"></canvas>
	</div>
</div>

<style>
	:global(*) {
		box-sizing: border-box;
	}

	:global(body) {
		margin: 0;
		padding: 0;
		font-family: sans-serif;
	}

	#root {
		display: flex;
		justify-content: left;
		align-items: flex-start;
		flex-direction: column;
		height: 100vh;
		width: 100vw;
	}

	#sigma-container {
		position: relative;
		width: 100%;
		height: 100%;
		background-color: #dddddd;
	}
	#background {
		position: absolute;
		z-index: 0;
		image-rendering: pixelated;
	}
</style>
