<template>
  <div id="sigma-container" style="width: 100vw; height: 100vh"></div>
  <div id="controls">
    <div class="input">
      <label for="zoom-in">Zoom in</label><button id="zoom-in">+</button>
    </div>
    <div class="input">
      <label for="zoom-out">Zoom out</label><button id="zoom-out">-</button>
    </div>
    <div class="input">
      <label for="zoom-reset">Reset zoom</label
      ><button id="zoom-reset">⊙</button>
    </div>
    <div class="input">
      <label for="labels-threshold">Labels threshold</label>
      <input id="labels-threshold" type="range" min="0" max="15" step="0.5" />
    </div>
  </div>
  <div id="search">
    <input
      type="search"
      id="search-input"
      list="suggestions"
      placeholder="Try searching for a node..."
    />
    <datalist id="suggestions"></datalist>
  </div>
</template>

<script lang="ts">
import Sigma from "sigma";
import Graph from "graphology";
import { parse } from "graphology-gexf/browser";
import { Vue } from "vue-class-component";
import chroma from "chroma-js";
import { v4 as uuid } from "uuid";
import NodeProgramFull from "./webgl/programs/node.full";
import getNodeProgramImage from "sigma/rendering/webgl/programs/node.image";
import FA2Layout from "graphology-layout-forceatlas2/worker";
import forceAtlas2 from "graphology-layout-forceatlas2";
import { Coordinates, EdgeDisplayData, NodeDisplayData } from "sigma/types";
import circlepack from "graphology-layout/circlepack";
import circular from "graphology-layout/circular";
import { layoutAnimate } from "@/lib/layoutAnimation";
import { drawHover } from "@/utils/canvas";
import { isNil } from "lodash";

fetch("http://10.109.92.74:8083/")
  .then((res) => res.json())
  .then((jsonObj) => {
    const graph = new Graph();
    graph.import(jsonObj);

    function colorize(str: string) {
      for (
        var i = 0, hash = 0;
        i < str.length;
        hash = str.charCodeAt(i++) + ((hash << 5) - hash)
      );
      let color = Math.floor(
        Math.abs(((Math.sin(hash) * 10000) % 1) * 16777216)
      ).toString(16);
      return "#" + Array(6 - color.length + 1).join("0") + color;
    }

    graph.forEachNode((node, attr) => {
      attr.color = chroma(colorize(attr["labels"][0])).hex();
      attr.label =
        attr["productName"] || attr["companyName"] || attr["shipName"];
      attr.size = attr["reorderLevel"] / 5;
      return attr;
    });

    circular.assign(graph);

    const sensibleSettings = forceAtlas2.inferSettings(graph);
    const layout = new FA2Layout(graph, {
      settings: sensibleSettings,
    });
    // layout.start();

    // Retrieve some useful DOM elements:
    const container = document.getElementById("sigma-container") as HTMLElement;
    const zoomInBtn = document.getElementById("zoom-in") as HTMLButtonElement;
    const zoomOutBtn = document.getElementById("zoom-out") as HTMLButtonElement;
    const zoomResetBtn = document.getElementById(
      "zoom-reset"
    ) as HTMLButtonElement;
    const labelsThresholdRange = document.getElementById(
      "labels-threshold"
    ) as HTMLInputElement;

    // Instanciate sigma:
    const renderer = new Sigma(graph, container, {
      minCameraRatio: 0.001,
      maxCameraRatio: 1000,
      nodeProgramClasses: {
        image: getNodeProgramImage(),
        circle: NodeProgramFull,
      },
      renderEdgeLabels: true,
    });

    const subtitleFields = [
      "productName",
      "categoryID",
      "discontinued",
      "labels",
      "productID",
      "quantityPerUnit",
      "reorderLevel",
      "supplierID",
      "unitPrice",
      "unitsInStock",
      "unitsOnOrder",
      "address",
      "city",
      "companyName",
      "contactName",
      "contactTitle",
      "country",
      "fax",
      "homePage",
      "phone",
      "postalCode",
      "region",
      "supplierID",
      "customerID",
      "employeeID",
      "freight",
      "orderDate",
      "orderID",
      "requiredDate",
      "shipAddress",
      "shipCity",
      "shipCountry",
      "shipName",
      "shipPostalCode",
      "shipRegion",
      "shipVia",
      "shippedDate",
    ];
    graph.forEachNode((node) =>
      graph.setNodeAttribute(
        node,
        "subtitles",
        subtitleFields.flatMap((subtitleField) => {
          const val = graph.getNodeAttributes(node)[subtitleField];
          return isNil(val)
            ? []
            : [
                `${subtitleField}: ${
                  typeof val === "number" ? val.toLocaleString() : val
                }`,
              ];
        })
      )
    );
    renderer.setSetting("hoverRenderer", (context, data, settings) =>
      drawHover(
        context,
        { ...renderer.getNodeDisplayData(data.key), ...data },
        settings
      )
    );

    const camera = renderer.getCamera();

    // Bind zoom manipulation buttons
    zoomInBtn.addEventListener("click", () => {
      camera.animatedZoom({ duration: 600 });
    });
    zoomOutBtn.addEventListener("click", () => {
      camera.animatedUnzoom({ duration: 600 });
    });
    zoomResetBtn.addEventListener("click", () => {
      camera.animatedReset({ duration: 600 });
    });

    // Bind labels threshold to range input
    labelsThresholdRange.addEventListener("input", () => {
      renderer.setSetting(
        "labelRenderedSizeThreshold",
        +labelsThresholdRange.value
      );
    });

    // Set proper range initial value:
    labelsThresholdRange.value =
      renderer.getSetting("labelRenderedSizeThreshold") + "";

    //
    // Create node (and edge) by click
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    //

    // When clicking on the stage, we add a new node and connect it to the closest node
    renderer.on(
      "clickStage",
      ({ event }: { event: { x: number; y: number } }) => {
        // // Sigma (ie. graph) and screen (viewport) coordinates are not the same.
        // // So we need to translate the screen x & y coordinates to the graph one by calling the sigma helper `viewportToGraph`
        // const coordForGraph = renderer.viewportToGraph({
        //   x: event.x,
        //   y: event.y,
        // });

        // // We create a new node
        // const node = {
        //   ...coordForGraph,
        //   size: 4,
        //   color: chroma.random().hex(),
        //   // type: "border",
        // };

        // // Searching the two closest nodes to auto-create an edge to it
        // const closestNodes = graph
        //   .nodes()
        //   .map((nodeId) => {
        //     const attrs = graph.getNodeAttributes(nodeId);
        //     const distance =
        //       Math.pow(node.x - attrs.x, 2) + Math.pow(node.y - attrs.y, 2);
        //     return { nodeId, distance };
        //   })
        //   .sort((a, b) => a.distance - b.distance)
        //   .slice(0, 2);

        // // We register the new node into graphology instance
        // const id = uuid();
        // graph.addNode(id, node);

        // // We create the edges
        // closestNodes.forEach((e) => graph.addEdge(id, e.nodeId));

        layoutAnimate(
          graph,
          circlepack(graph, {
            hierarchyAttributes: ["labels"],
            scale: 0.02,
          })
        );
      }
    );

    //
    // highlight and search
    // ~~~~~~~~~~~~~~~~~~~~
    //

    const searchInput = document.getElementById(
      "search-input"
    ) as HTMLInputElement;
    const searchSuggestions = document.getElementById(
      "suggestions"
    ) as HTMLDataListElement;

    // Type and declare internal state:
    interface State {
      hoveredNode?: string;
      searchQuery: string;

      // State derived from query:
      selectedNode?: string;
      suggestions?: Set<string>;

      // State derived from hovered node:
      hoveredNeighbors?: Set<string>;

      isDragging?: boolean;
    }
    const state: State = { searchQuery: "", isDragging: false };

    // Feed the datalist autocomplete values:
    searchSuggestions.innerHTML = graph
      .nodes()
      .map(
        (node) =>
          `<option value="${graph.getNodeAttribute(node, "label")}"></option>`
      )
      .join("\n");

    // Actions:
    function setSearchQuery(query: string) {
      state.searchQuery = query;

      if (searchInput.value !== query) searchInput.value = query;

      if (query) {
        const lcQuery = query.toLowerCase();
        const suggestions = graph
          .nodes()
          .map((n) => ({
            id: n,
            label: graph.getNodeAttribute(n, "label") as string,
          }))
          .filter(({ label }) => label.toLowerCase().includes(lcQuery));

        // If we have a single perfect match, them we remove the suggestions, and
        // we consider the user has selected a node through the datalist
        // autocomplete:
        if (suggestions.length === 1 && suggestions[0].label === query) {
          state.selectedNode = suggestions[0].id;
          state.suggestions = undefined;

          // Move the camera to center it on the selected node:
          const nodePosition = renderer.getNodeDisplayData(
            state.selectedNode
          ) as Coordinates;
          renderer.getCamera().animate(nodePosition, {
            duration: 500,
          });
        }
        // Else, we display the suggestions list:
        else {
          state.selectedNode = undefined;
          state.suggestions = new Set(suggestions.map(({ id }) => id));
        }
      }
      // If the query is empty, then we reset the selectedNode / suggestions state:
      else {
        state.selectedNode = undefined;
        state.suggestions = undefined;
      }

      // Refresh rendering:
      renderer.refresh();
    }
    function setHoveredNode(node?: string) {
      if (node) {
        state.hoveredNode = node;
        state.hoveredNeighbors = new Set(graph.neighbors(node));
      } else {
        state.hoveredNode = undefined;
        state.hoveredNeighbors = undefined;
      }

      // Refresh rendering:
      renderer.refresh();
    }

    // Bind search input interactions:
    searchInput.addEventListener("input", () => {
      setSearchQuery(searchInput.value || "");
    });
    searchInput.addEventListener("blur", () => {
      setSearchQuery("");
    });

    // Bind graph interactions:
    renderer.on("enterNode", ({ node }) => {
      if (!state.isDragging) {
        setHoveredNode(node);
      }
    });
    renderer.on("leaveNode", () => {
      if (!state.isDragging) {
        setHoveredNode(undefined);
      }
    });

    // Render nodes accordingly to the internal state:
    // 1. If a node is selected, it is highlighted
    // 2. If there is query, all non-matching nodes are greyed
    // 3. If there is a hovered node, all non-neighbor nodes are greyed
    renderer.setSetting("nodeReducer", (node, data) => {
      const res: Partial<NodeDisplayData> = { ...data };

      if (
        state.hoveredNeighbors &&
        !state.hoveredNeighbors.has(node) &&
        state.hoveredNode !== node
      ) {
        res.label = "";
        res.color = "#f6f6f6";
      }

      if (state.selectedNode === node) {
        res.highlighted = true;
      } else if (state.suggestions && !state.suggestions.has(node)) {
        res.label = "";
        res.color = "#f6f6f6";
      }

      return res;
    });

    // Render edges accordingly to the internal state:
    // 1. If a node is hovered, the edge is hidden if it is not connected to the
    //    node
    // 2. If there is a query, the edge is only visible if it connects two
    //    suggestions
    renderer.setSetting("edgeReducer", (edge, data) => {
      const res: Partial<EdgeDisplayData> = { ...data };

      if (state.hoveredNode && !graph.hasExtremity(edge, state.hoveredNode)) {
        res.hidden = true;
      }

      if (
        state.suggestions &&
        (!state.suggestions.has(graph.source(edge)) ||
          !state.suggestions.has(graph.target(edge)))
      ) {
        res.hidden = true;
      }

      return res;
    });

    //
    // Drag'n'drop feature
    // ~~~~~~~~~~~~~~~~~~~
    //

    // State for drag'n'drop
    let draggedNode: string | null = null;
    state.isDragging = false;

    // On mouse down on a node
    //  - we enable the drag mode
    //  - save in the dragged node in the state
    //  - highlight the node
    //  - disable the camera so its state is not updated
    renderer.on("downNode", (e) => {
      state.isDragging = true;
      layout.stop();
      draggedNode = e.node;
      graph.setNodeAttribute(draggedNode, "highlighted", true);
    });

    // On mouse move, if the drag mode is enabled, we change the position of the draggedNode
    renderer.getMouseCaptor().on("mousemovebody", (e) => {
      if (!state.isDragging || !draggedNode) return;

      // Get new position of node
      const pos = renderer.viewportToGraph(e);

      graph.setNodeAttribute(draggedNode, "x", pos.x);
      graph.setNodeAttribute(draggedNode, "y", pos.y);

      // Prevent sigma to move camera:
      e.preventSigmaDefault();
      e.original.preventDefault();
      e.original.stopPropagation();
    });

    // On mouse up, we reset the autoscale and the dragging mode
    renderer.getMouseCaptor().on("mouseup", () => {
      if (draggedNode) {
        graph.removeNodeAttribute(draggedNode, "highlighted");
      }
      state.isDragging = false;
      // layout.start();
      draggedNode = null;
    });

    // Disable the autoscale at the first down interaction
    renderer.getMouseCaptor().on("mousedown", () => {
      if (!renderer.getCustomBBox()) renderer.setCustomBBox(renderer.getBBox());
    });
  });
export default class GraphComponent extends Vue {}
</script>

<style>
body {
  font-family: sans-serif;
}

html,
body,
#sigma-container {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
}

#controls {
  position: absolute;
  right: 1em;
  top: 1em;
  text-align: right;
}

.input {
  position: relative;
  display: inline-block;
  vertical-align: middle;
}

.input:not(:hover) label {
  display: none;
}

.input label {
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  background: black;
  color: white;
  padding: 0.2em;
  border-radius: 2px;
  margin-top: 0.3em;
  font-size: 0.8em;
  white-space: nowrap;
}

.input button {
  width: 2.5em;
  height: 2.5em;
  display: inline-block;
  text-align: center;
  background: white;
  outline: none;
  border: 1px solid dimgrey;
  border-radius: 2px;
  cursor: pointer;
}

#search {
  position: absolute;
  right: 1em;
  top: 4em;
}
</style>
