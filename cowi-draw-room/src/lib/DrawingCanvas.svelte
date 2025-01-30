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
          generateGrid();
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
          polygon[draggedVertex] = point;
          polygon = [...polygon];
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
              saveState();
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
              saveState();
              return;
          }
      }

      polygon = [...polygon, point];
      saveState();
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
      saveState();
  }

  function removeCircle(point, event) {
      if (event) {
          event.stopPropagation();
      }
      placedCircles = placedCircles.filter(p => 
          p.x !== point.x || p.y !== point.y
      );
      saveState();
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
      history = [];
  }

  let mousePosition = { x: 0, y: 0 };

  $: if (gridSize && polygon.length >= 3 && !isDrawing) {
      generateGrid();
  }
</script>

<svelte:window on:keydown={handleKeyboard}/>

<div class="app">
    <div class="controls-panel">
        <div class="control-group">
            <label>
                <input type="radio" bind:group={mode} value="polygon" disabled={isDrawing} />
                Draw Polygon
            </label>
            <label>
                <input type="radio" bind:group={mode} value="circle" disabled={isDrawing || polygon.length < 3} />
                Place Circles
            </label>
            <button class="control-button" on:click={resetDrawing}>Reset</button>
        </div>

        <div class="control-group">
            <label class:disabled={placedCircles.length > 0}>
                Grid: {gridSize}px
                <input type="range" min="5" max="100" bind:value={gridSize} disabled={placedCircles.length > 0} />
            </label>
            
            <label class:disabled={placedCircles.length > 0}>
                Points: {smallCircleRadius}px
                <input type="range" min="1" max="5" bind:value={smallCircleRadius} disabled={placedCircles.length > 0} />
            </label>
            
            <label>
                Circles: {largeCircleRadius}px
                <input type="range" min="5" max="100" bind:value={largeCircleRadius} />
            </label>
        </div>
        
        {#if !isDrawing && polygon.length >= 3}
            <div class="control-group">
                <button class="control-button" on:click={removeLastPoint}>Remove Point</button>
                <button class="control-button" on:click={addNewPoint}>Add Point</button>
            </div>
        {/if}
    </div>

    <svg
        bind:this={svgElement}
        class:circle-mode={mode === 'circle'}
        on:click={handleClick}
        on:mousemove={handleMouseMove}
        on:mouseup={stopDragging}
        on:mouseleave={stopDragging}
    >
        {#if polygon.length > 0}
            <path
                d={`M ${polygon.map(p => `${p.x},${p.y}`).join(' L ')}${!isDrawing ? ' Z' : ''}`}
                fill={!isDrawing ? "#f8f9fa" : "none"}
                stroke="#212529"
                stroke-width="2"
            />
            
            {#if isDrawing && polygon.length > 0}
                <line
                    x1={polygon[polygon.length - 1].x}
                    y1={polygon[polygon.length - 1].y}
                    x2={mousePosition.x}
                    y2={mousePosition.y}
                    stroke="#212529"
                    stroke-width="1"
                    stroke-dasharray="5,5"
                />
            {/if}
            
            {#each polygon as point, i}
                <circle
                    cx={point.x}
                    cy={point.y}
                    r="6"
                    fill="#212529"
                    class="vertex"
                    on:mousedown={(e) => startDragging(i, e)}
                />
            {/each}
        {/if}
        
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
                    fill="#6c757d"
                    class="grid-point"
                    pointer-events="none"
                />
            </g>
        {/each}
        
        {#if mode === 'circle' && previewPoint}
            <circle
                cx={previewPoint.x}
                cy={previewPoint.y}
                r={largeCircleRadius}
                fill="#0d6efd"
                opacity="0.3"
                pointer-events="none"
            />
        {/if}
        
        {#each placedCircles as point}
            <g>
                <circle
                    cx={point.x}
                    cy={point.y}
                    r={largeCircleRadius}
                    fill="#0d6efd"
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
</div>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        overflow: hidden;
        background-color: #ffffff;
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
    }

    :global(*) {
        box-sizing: border-box;
        margin: 0;
        padding: 0;
    }

    .app {
        width: 100%;
        height: 100%;
        position: fixed;
        top: 0;
        left: 0;
        margin: 0;
        padding: 0;
        overflow: hidden;
    }

    svg {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        display: block;
        margin: 0;
        padding: 0;
    }

    .controls-panel {
        position: fixed;
        top: 1rem;
        left: 1rem;
        background: rgba(255, 255, 255, 0.95);
        padding: 1rem;
        border-radius: 8px;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
        z-index: 1000;
        display: flex;
        flex-direction: column;
        gap: 1rem;
        max-width: 300px;
    }

    .control-group {
        display: flex;
        flex-direction: column;
        gap: 0.75rem;
        padding-bottom: 0.75rem;
        border-bottom: 1px solid #dee2e6;
    }

    .control-group:last-child {
        border-bottom: none;
        padding-bottom: 0;
    }

    .control-button {
        padding: 0.5rem 1rem;
        background: #f8f9fa;
        border: 1px solid #dee2e6;
        border-radius: 4px;
        cursor: pointer;
        font-size: 0.875rem;
        transition: all 0.2s ease;
    }

    .control-button:hover {
        background: #e9ecef;
        border-color: #ced4da;
    }

    label {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        font-size: 0.875rem;
        color: #495057;
        user-select: none;
    }

    input[type="range"] {
        flex: 1;
    }

    .circle-mode {
        cursor: default;
    }

    .vertex {
        cursor: move;
        transition: fill 0.2s ease;
    }

    .vertex:hover {
        fill: #495057;
    }

    .grid-point-hitbox {
        cursor: pointer;
    }

    .placed-circle {
        cursor: pointer;
        transition: opacity 0.2s ease;
    }

    .placed-circle:hover {
        opacity: 0.7;
    }

    .remove-indicator {
        font-size: 16px;
        fill: white;
        user-select: none;
    }

    .disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }

    .disabled input {
        cursor: not-allowed;
    }
</style>