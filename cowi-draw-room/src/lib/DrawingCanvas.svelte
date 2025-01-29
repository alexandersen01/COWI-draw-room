<script>
    import { onMount } from 'svelte';
    import { polygonContains, polygonHull, polygonCentroid } from 'd3-polygon';
  
    let polygon = [];
    let gridPoints = [];
    let placedCircles = [];
    let gridSize = 20;
    let smallCircleRadius = 2;
    let largeCircleRadius = 30;
    let isDrawing = false;
    let firstPoint = null;
    let svgElement;
    let mode = 'polygon';
    let previewPoint = null;
    let isDragging = false;
    let draggedVertex = null;
    let history = [];
    let currentState = {
      polygon: [],
      placedCircles: []
    };

    function handleKeyboard(event) {
        if ((event.ctrlKey || event.metaKey) && event.key === 'z') {
            event.preventDefault();
            undo();
        }
    }

    function saveState() {
        history.push(JSON.stringify({
            polygon: polygon,
            placedCircles: placedCircles
        }));
    }
    
    function undo() {
        if (history.length > 0) {
            const previousState = JSON.parse(history.pop());
            polygon = previousState.polygon;
            placedCircles = previousState.placedCircles;
            generateGrid(); // Regenerate grid after undoing
        }
    }
    
    function getMousePosition(event) {
      const pt = svgElement.createSVGPoint();
      pt.x = event.clientX;
      pt.y = event.clientY;
      const svgPoint = pt.matrixTransform(svgElement.getScreenCTM().inverse());
      return { x: svgPoint.x, y: svgPoint.y };
    }
    
    function startDragging(index, event) {
      if (mode !== 'polygon' || isDrawing) return;
      event.stopPropagation();
      isDragging = true;
      draggedVertex = index;
    }
    
    function handleMouseMove(event) {
      const point = getMousePosition(event);
      
      if (isDragging && draggedVertex !== null) {
        // Update the dragged vertex position
        polygon[draggedVertex] = point;
        polygon = [...polygon]; // Trigger reactivity
        generateGrid();
      } else if (isDrawing) {
        mousePosition = point;
      }
    }
    
    function stopDragging() {
      isDragging = false;
      draggedVertex = null;
    }
  
    function handleClick(event) {
        if (mode !== 'polygon' || isDragging) return;
        
        const point = getMousePosition(event);
        
        if (!isDrawing) {
            if (polygon.length === 0) {
                firstPoint = point;
                polygon = [point];
                isDrawing = true;
                saveState(); // Save state after starting new polygon
            }
            return;
        }

        if (firstPoint) {
            const dx = point.x - firstPoint.x;
            const dy = point.y - firstPoint.y;
            const distanceToStart = Math.sqrt(dx * dx + dy * dy);
            
            if (distanceToStart < 20 && polygon.length >= 2) {
                polygon = [...polygon, firstPoint];
                isDrawing = false;
                generateGrid();
                saveState(); // Save state after completing polygon
                return;
            }
        }

        polygon = [...polygon, point];
        saveState(); // Save state after adding vertex
    }
  
    function generateGrid() {
      if (polygon.length < 3) return;
      
      const polygonPoints = polygon.map(p => [p.x, p.y]);
      
      const xCoords = polygonPoints.map(p => p[0]);
      const yCoords = polygonPoints.map(p => p[1]);
      const minX = Math.min(...xCoords);
      const maxX = Math.max(...xCoords);
      const minY = Math.min(...yCoords);
      const maxY = Math.max(...yCoords);
      
      const newGridPoints = [];
      
      for (let x = Math.floor(minX); x <= Math.ceil(maxX); x += gridSize) {
        for (let y = Math.floor(minY); y <= Math.ceil(maxY); y += gridSize) {
          if (polygonContains(polygonPoints, [x, y])) {
            newGridPoints.push({ x, y });
          }
        }
      }
      
      gridPoints = newGridPoints;
      
      // Update placed circles - remove any that are now outside the polygon
      placedCircles = placedCircles.filter(circle => {
        return polygonContains(polygonPoints, [circle.x, circle.y]);
      });
    }
  
    function handleGridPointClick(point, event) {
      if (mode !== 'circle') return;
      event.stopPropagation();
      toggleCircle(point);
    }
  
    function handleGridPointMouseEnter(point) {
      if (mode === 'circle') {
        previewPoint = point;
      }
    }
  
    function handleGridPointMouseLeave() {
      previewPoint = null;
    }
  
    function toggleCircle(point) {
        if (mode !== 'circle') return;
        placedCircles = [...placedCircles, point];
        saveState(); // Save state after placing circle
    }
  
    function removeCircle(point, event) {
        if (event) {
            event.stopPropagation();
        }
        placedCircles = placedCircles.filter(p => 
            p.x !== point.x || p.y !== point.y
        );
        saveState(); // Save state after removing circle
    }
  
    function removeLastPoint() {
      if (polygon.length > 3) {
        polygon = polygon.slice(0, -1);
        generateGrid();
      }
    }
  
    function addNewPoint() {
      if (polygon.length < 3) return;
      
      const lastPoint = polygon[polygon.length - 1];
      const prevPoint = polygon[polygon.length - 2];
      
      const midPoint = {
        x: (lastPoint.x + prevPoint.x) / 2,
        y: (lastPoint.y + prevPoint.y) / 2
      };
      
      polygon = [...polygon.slice(0, -1), midPoint, lastPoint];
      generateGrid();
    }
  
    function resetDrawing() {
        polygon = [];
        gridPoints = [];
        placedCircles = [];
        isDrawing = false;
        firstPoint = null;
        previewPoint = null;
        isDragging = false;
        draggedVertex = null;
        history = []; // Clear history when resetting
    }
  
    let mousePosition = { x: 0, y: 0 };
  
    $: if (gridSize && polygon.length >= 3 && !isDrawing) {
      generateGrid();
    }
</script>
<svelte:window on:keydown={handleKeyboard}/>
<div class="controls">
    <div class="mode-switcher">
        <label>
            <input
                type="radio"
                bind:group={mode}
                value="polygon"
                disabled={isDrawing}
            />
            Draw/Edit Polygon
        </label>
        <label>
            <input
                type="radio"
                bind:group={mode}
                value="circle"
                disabled={isDrawing || polygon.length < 3}
            />
            Place Circles
        </label>
        <button on:click={resetDrawing}>Reset</button>
    </div>

    <div class="sliders">
        <label class:disabled={placedCircles.length > 0}>
            Grid Size:
            <input 
                type="range" 
                min="5" 
                max="100" 
                bind:value={gridSize}
                disabled={placedCircles.length > 0}
            />
            {gridSize}px
        </label>
        
        <label class:disabled={placedCircles.length > 0}>
            Grid Point Size:
            <input 
                type="range" 
                min="1" 
                max="5" 
                bind:value={smallCircleRadius}
                disabled={placedCircles.length > 0}
            />
            {smallCircleRadius}px
        </label>
        
        <label>
            Placed Circle Size:
            <input 
                type="range" 
                min="5" 
                max="100" 
                bind:value={largeCircleRadius}
            />
            {largeCircleRadius}px
        </label>
    </div>
    
    {#if !isDrawing && polygon.length >= 3}
        <div class="polygon-controls">
            <button on:click={removeLastPoint}>Remove Last Point</button>
            <button on:click={addNewPoint}>Add New Point</button>
        </div>
    {/if}
</div>

<svg
    bind:this={svgElement}
    width="800"
    height="600"
    on:click={handleClick}
    on:mousemove={handleMouseMove}
    on:mouseup={stopDragging}
    on:mouseleave={stopDragging}
    class:circle-mode={mode === 'circle'}
>
    <!-- Draw polygon -->
    {#if polygon.length > 0}
        <path
            d={`M ${polygon.map(p => `${p.x},${p.y}`).join(' L ')}${!isDrawing ? ' Z' : ''}`}
            fill={!isDrawing ? "#f0f0f0" : "none"}
            stroke="black"
            stroke-width="2"
        />
        
        <!-- Preview line while drawing -->
        {#if isDrawing && polygon.length > 0}
            <line
                x1={polygon[polygon.length - 1].x}
                y1={polygon[polygon.length - 1].y}
                x2={mousePosition.x}
                y2={mousePosition.y}
                stroke="black"
                stroke-width="1"
                stroke-dasharray="5,5"
            />
        {/if}
        
        <!-- Draw vertex points -->
        {#each polygon as point, i}
            <circle
                cx={point.x}
                cy={point.y}
                r="6"
                fill="black"
                class="vertex"
                on:mousedown={(e) => startDragging(i, e)}
                style="cursor: move;"
            />
        {/each}
    {/if}
    
    <!-- Draw grid points -->
    {#each gridPoints as point}
        <g>
            <circle
                cx={point.x}
                cy={point.y}
                r="10"
                fill="transparent"
                class="grid-point-hitbox"
                on:click={(e) => handleGridPointClick(point, e)}
                on:mouseenter={() => handleGridPointMouseEnter(point)}
                on:mouseleave={handleGridPointMouseLeave}
            />
            <circle
                cx={point.x}
                cy={point.y}
                r={smallCircleRadius}
                fill="gray"
                class="grid-point"
                pointer-events="none"
            />
        </g>
    {/each}
    
    <!-- Preview circle -->
    {#if mode === 'circle' && previewPoint}
        <circle
            cx={previewPoint.x}
            cy={previewPoint.y}
            r={largeCircleRadius}
            fill="blue"
            opacity="0.3"
            pointer-events="none"
        />
    {/if}
    
    <!-- Draw placed circles -->
    {#each placedCircles as point}
        <g>
            <circle
                cx={point.x}
                cy={point.y}
                r={largeCircleRadius}
                fill="blue"
                opacity="0.5"
                on:click={(e) => removeCircle(point, e)}
                class="placed-circle"
            />
            {#if mode === 'circle'}
                <text
                    x={point.x}
                    y={point.y}
                    text-anchor="middle"
                    dominant-baseline="middle"
                    class="remove-indicator"
                    pointer-events="none"
                >
                    ×
                </text>
            {/if}
        </g>
    {/each}
</svg>

<style>
    .controls {
        margin-bottom: 1rem;
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .mode-switcher {
        display: flex;
        gap: 1rem;
        align-items: center;
    }

    .sliders {
        display: flex;
        gap: 1rem;
        align-items: center;
    }

    .polygon-controls {
        display: flex;
        gap: 1rem;
    }
    
    svg {
        border: 1px solid #ccc;
        background: #f9f9f9;
    }

    .circle-mode {
        cursor: default;
    }

    .circle-mode .grid-point {
        cursor: pointer;
    }

    .grid-point-hitbox:hover + .grid-point {
        fill: #666;
    }

    .grid-point-hitbox {
        cursor: pointer;
    }

    .placed-circle {
        cursor: pointer;
    }

    .placed-circle:hover {
        opacity: 0.7;
    }

    .remove-indicator {
        font-size: 16px;
        fill: white;
        user-select: none;
    }

    .vertex {
        cursor: move;
    }

    .vertex:hover {
        fill: #444;
    }

    label {
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    button {
        padding: 0.5rem 1rem;
        background: #e0e0e0;
        border: 1px solid #ccc;
        border-radius: 4px;
        cursor: pointer;
    }

    button:hover {
        background: #d0d0d0;
    }

    .disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }

    .disabled input {
        cursor: not-allowed;
    }
</style>