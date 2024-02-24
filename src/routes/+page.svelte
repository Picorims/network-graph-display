<div id="root">
    <input type="file" name="file" id="file-input" accept=".json" on:change={loadGraph}>
    <input type="file" name="file" id="bgnd-input" accept=".jpeg, .jpg, .png" on:change={loadBackground}>
    <span>Sum: <span>{sum}</span></span>
    <div id="sigma-container" use:init>
        <div id="background"></div>
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
        justify-content: center;
        align-items: center;
        flex-direction: column;
        height: 100vh;
        width: 100vw;
    }

    #sigma-container {
        width: 100%;
        height: 100%;
        background-color: #dddddd;
    }
    #background {
        width: 100%;
        height: 100%;
        position: absolute;
        top: 0;
        left: 0;
        z-index: -1;
        background-size: contain;
        background-position: center;
        background-repeat: no-repeat;
        image-rendering: pixelated;
    }
</style>

<script lang="ts">
    /**
     * This is a minimal example of sigma. You can use it as a base to write new
     * examples, or reproducible test cases for new issues, for instance.
     */

    import Graph from "graphology";
    import Sigma from "sigma";
	import dijkstra from "graphology-shortest-path/dijkstra";
	import type { EdgeDisplayData } from "sigma/types";

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
    
    function init(node: HTMLElement) {
        container = document.getElementById("sigma-container");
        if (!container) throw 'No container found';
        console.log(container);

        graph = new Graph();
    
        graph.addNode("John", { x: 0, y: 10, size: 5, label: "John", color: "blue" });
        graph.addNode("Mary", { x: 10, y: 0, size: 3, label: "Mary", color: "red" });
    
        graph.addEdge("John", "Mary");
    
        // eslint-disable-next-line @typescript-eslint/no-unused-vars
        renderer = new Sigma(graph, container, {
            maxCameraRatio: 4,
            minCameraRatio: 0.01,
            renderEdgeLabels: true,
            defaultNodeColor: "#00000022"

        });

        renderer.on("clickNode", (event) => {
            console.log(event);
            if (from === null) {
                from = event.node;
            } else if (to === null) {
                to = event.node;
    
                console.log(graph.nodes())
                const path = dijkstra.bidirectional(graph, from, to);
    
                if (path === null) {
                    alert("No path found");
                } else {
                    console.log(path);

                    state.path = path;
                    sum = 0;
                    renderer?.refresh();
                }
    
                from = to = null;
            } 
        });

        renderer.setSetting("edgeReducer", (edge, data) => {
            const res: Partial<EdgeDisplayData> = {...data};
            console.log(data, edge);
            const source = graph.source(edge);
            const target = graph.target(edge);
            let index = state.path.indexOf(source);
            if (index !== -1) {
                if (state.path[index + 1] && state.path[index + 1] === target || state.path[index - 1] && state.path[index - 1] === target) {
                    res.size = 3;
                    sum += data.weight;
                }
            }

            return res;
        });
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
        graph = new Graph({ multi: graphJSON.multigraph, type: graphJSON.directed ? "directed" : "undirected"});

        for (const node of graphJSON.nodes) {
            graph.addNode(node.id, { x: node.x, y: node.y * -1, label: node.id.toString(), size: 5 });
        }

        for (const link of graphJSON.links) {
            graph.addEdge(link.source, link.target, { weight: link.weight, color: link.color, label: link.weight.toString() });
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
            container.style.backgroundImage = `url(${event.target?.result})`;
        };
        reader.readAsDataURL(file);
    }

</script>