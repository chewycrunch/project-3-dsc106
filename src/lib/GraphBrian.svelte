<script lang="ts">
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  // Component props
  export let width = 800;
  export let height = 400;
  export let margin = { top: 40, right: 60, bottom: 40, left: 60 };

  // Type definitions
  interface ProcessedDataPoint {
    minute: number;
    day: number;
    avgTemp: number;
    light: number;
    estrus: number;
  }

  type DataType = 'female' | 'male' | 'both';

  let svg: SVGElement;
  let brushSvg: SVGElement;
  let data: any[] = [];
  let processedData: ProcessedDataPoint[] = [];
  let useFahrenheit = false;
  let dataType: DataType = 'female';
  let showEstrus = true;  // New state variable for estrus toggle
  let zoom: d3.ZoomBehavior<SVGElement, unknown>;

  // Temperature conversion functions
  function toFahrenheit(celsius: number): number {
    return (celsius * 9/5) + 32;
  }

  function toCelsius(fahrenheit: number): number {
    return (fahrenheit - 32) * 5/9;
  }

  // Get temperature display value based on current unit
  function getDisplayTemp(temp: number): number {
    return useFahrenheit ? toFahrenheit(temp) : temp;
  }

  // Get temperature display unit
  function getTempUnit(): string {
    return useFahrenheit ? '°F' : '°C';
  }

  // Get y-axis domain based on current unit
  function getYDomain(): [number, number] {
    return useFahrenheit 
      ? [toFahrenheit(35.25), toFahrenheit(39)]
      : [35.25, 39];
  }

  // Toggle temperature unit
  function toggleTempUnit() {
    useFahrenheit = !useFahrenheit;
    isBrushed = false;
    selectedData = [];
    createVisualization();
    createBrushVisualization();
  }

  // Format x-axis ticks based on scale range
  function formatXAxis(scale: d3.ScaleLinear<number, number>): d3.Axis<d3.NumberValue> {
    const domain = scale.domain();
    const range = domain[1] - domain[0];
    
    // If selection is less than 1 day, show hours
    if (range <= 1) {
      return d3.axisBottom(scale)
        .ticks(6)
        .tickFormat(d => {
          const dNum = Number(d);
          const totalHours = dNum * 24;
          const hours = Math.floor(totalHours);
          const minutes = Math.round((totalHours - hours) * 60);
          return `${hours}:${minutes.toString().padStart(2, '0')}`;
        });
    } 
    // Otherwise show days
    else {
      return d3.axisBottom(scale)
        .ticks(6)
        .tickFormat(d => `Day ${Math.floor(Number(d) + 1)}`);
    }
  }

  // Process the data
  async function loadAndProcessData() {
    const response = await fetch('/mouse_data.csv');
    const csvText = await response.text();
    const rows = d3.csvParse(csvText);
    
    // Get temperature columns based on selected data type
    const getTempCols = (type: DataType) => {
      switch (type) {
        case 'female':
          return Array.from({length: 13}, (_, i) => `fem_temp_${i + 1}`);
        case 'male':
          return Array.from({length: 13}, (_, i) => `male_temp_${i + 1}`);
        case 'both':
          return [
            ...Array.from({length: 13}, (_, i) => `fem_temp_${i + 1}`),
            ...Array.from({length: 13}, (_, i) => `male_temp_${i + 1}`)
          ];
      }
    };
    
    // Group data into 2.5-minute windows
    const windowSize = 2.5; // 2.5 minutes per window
    const groupedData = new Map<number, number[]>();
    
    rows.forEach((d: any, i: number) => {
      const windowIndex = Math.floor(i / windowSize);
      const tempCols = getTempCols(dataType);
      const temps = tempCols.map(col => +d[col]);
      const avgTemp = d3.mean(temps) || 0;
      
      if (!groupedData.has(windowIndex)) {
        groupedData.set(windowIndex, []);
      }
      groupedData.get(windowIndex)?.push(avgTemp);
    });
    
    // Calculate average for each window and create processed data
    processedData = Array.from(groupedData.entries()).map(([windowIndex, temps]) => {
      const minute = windowIndex * windowSize;
      const day = Math.floor(minute / 1440);
      const minuteOfDay = minute % 1440;
      const light = minuteOfDay < 720 ? 1 : 0;
      const estrus = (day % 4 === 2) ? 1 : 0;
      
      return {
        minute,
        day: minute / 1440, // Convert to days for x-axis
        avgTemp: d3.mean(temps) || 0,
        light,
        estrus
      };
    });

    // Reset brush selection
    currentBrushSelection = null;

    // Update visualization with new data
    createVisualization();
    createBrushVisualization();
  }

  function createVisualization() {
    if (!processedData.length) return;

    // Clear previous visualization
    d3.select(svg).selectAll("*").remove();

    const innerWidth = width - margin.left - margin.right;
    const innerHeight = height - margin.top - margin.bottom;

    // Create scales - will be updated by brush
    const xDomain = currentBrushSelection 
      ? currentBrushSelection 
      : [0, 14]; // default to full range
    
    const xScale = d3.scaleLinear()
      .domain(xDomain)
      .range([0, innerWidth]);

    const yScale = d3.scaleLinear()
      .domain(getYDomain())
      .range([innerHeight, 0]);

    // Create tick values for y-axis
    const domain = getYDomain();
    const tickCount = 6; // Number of ticks including boundaries
    const step = (domain[1] - domain[0]) / (tickCount - 1);
    const tickValues = Array.from({ length: tickCount }, (_, i) => 
      +(domain[0] + (step * i)).toFixed(1)
    );

    // Create SVG group
    const g = d3.select(svg)
      .attr('width', width)
      .attr('height', height)
      .attr('viewBox', `0 0 ${width} ${height}`)
      .attr('preserveAspectRatio', 'xMidYMid meet')
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    // Create background group for light/dark periods
    const bgGroup = g.append('g')
      .attr('class', 'background');

    // Add light/dark period shading
    for (let day = 0; day < 14; day++) {
      // Dark period (gray)
      bgGroup.append('rect')
        .attr('x', xScale(Math.max(day, xDomain[0])))
        .attr('y', 0)
        .attr('width', xScale(Math.min(day + 0.5, xDomain[1])) - xScale(Math.max(day, xDomain[0])))
        .attr('height', innerHeight)
        .attr('fill', 'gray')
        .attr('opacity', 0.1);

      // Light period (yellow)
      bgGroup.append('rect')
        .attr('x', xScale(Math.max(day + 0.5, xDomain[0])))
        .attr('y', 0)
        .attr('width', xScale(Math.min(day + 1, xDomain[1])) - xScale(Math.max(day + 0.5, xDomain[0])))
        .attr('height', innerHeight)
        .attr('fill', 'yellow')
        .attr('opacity', 0.1);
    }

    // Add estrus day highlighting only if enabled and female mice selected
    if (showEstrus && dataType === 'female') {
      const estrusGroup = g.append('g')
        .attr('class', 'estrus');

      [2, 6, 10].forEach(day => {
        if (day + 1 >= xDomain[0] && day <= xDomain[1]) {
          estrusGroup.append('rect')
            .attr('x', xScale(Math.max(day, xDomain[0])))
            .attr('y', 0)
            .attr('width', xScale(Math.min(day + 1, xDomain[1])) - xScale(Math.max(day, xDomain[0])))
            .attr('height', innerHeight)
            .attr('fill', '#FCCFCE') // Brighter red
            .attr('opacity', 1)
            .style('mix-blend-mode', 'multiply');
        }
      });
    }

    // Filter data points to those in the current domain
    const visibleData = processedData.filter(d => 
      d.day >= xDomain[0] && d.day <= xDomain[1]
    );

    // Create line generator with proper typing
    const line = d3.line<ProcessedDataPoint>()
      .x(d => xScale(d.day))
      .y(d => yScale(getDisplayTemp(d.avgTemp)))
      .curve(d3.curveMonotoneX);

    // Add the temperature line
    vizGroup.append('path')
      .datum(processedData)
      .attr('fill', 'none')
      .attr('stroke', '#1f77b4')
      .attr('stroke-width', 1.5)
      .attr('d', line);

    // Add title (outside of vizGroup)
    g.append('text')
      .attr('class', 'title')
      .attr('x', innerWidth / 2)
      .attr('y', -20)
      .attr('text-anchor', 'middle')
      .style('font-size', '16px')
      .text(titleText);

    // Add tooltip
    const tooltip = d3.select('body')
      .append('div')
      .attr('class', 'mouse-tooltip')
      .style('position', 'absolute')
      .style('background-color', 'white')
      .style('padding', '5px')
      .style('border', '1px solid #ddd')
      .style('border-radius', '3px')
      .style('pointer-events', 'none')
      .style('opacity', 0);

    // Add interactive dots
    vizGroup.selectAll('.dot')
      .data(processedData)
      .enter()
      .append('circle')
      .attr('class', 'dot')
      .attr('cx', d => xScale(d.day))
      .attr('cy', d => yScale(getDisplayTemp(d.avgTemp)))
      .attr('r', 2)
      .style('fill', '#1f77b4')
      .style('opacity', 0)
      .on('mouseover', function(event, d) {
        d3.select(this)
          .style('opacity', 1)
          .attr('r', 4);
        
        tooltip.transition()
          .duration(200)
          .style('opacity', .9);
        
        const day = Math.floor(d.day) + 1;
        const hour = Math.floor((d.day % 1) * 24);
        const minute = Math.floor(((d.day % 1) * 24 % 1) * 60);
        
        tooltip.html(
          `Day ${day} - ${hour}:${minute.toString().padStart(2, '0')}<br/>
           Temperature: ${getDisplayTemp(d.avgTemp).toFixed(2)}${getTempUnit()}<br/>
           Period: ${d.light ? 'Lights Off' : 'Lights On'}`
          + (dataType === 'female' ? `<br/>Estrus: ${d.estrus ? 'Yes' : 'No'}` : '')
        )
          .style('left', (event.pageX + 10) + 'px')
          .style('top', (event.pageY - 28) + 'px');
      })
      .on('mouseout', function() {
        d3.select(this)
          .style('opacity', 0)
          .attr('r', 2);
        
        tooltip.transition()
          .duration(500)
          .style('opacity', 0);
      });

    // Set up zoom behavior
    zoom = d3.zoom<SVGElement, unknown>()
      .scaleExtent([1, 10])
      .on('zoom', (event) => {
        // Update the visualization group transform
        vizGroup.attr('transform', event.transform);

        // Create new scales based on the zoom transform
        const newXScale = event.transform.rescaleX(xScale);
        const newYScale = event.transform.rescaleY(yScale);

        // Update the axes with new scales
        xAxisGroup.call(d3.axisBottom(newXScale));
        yAxisGroup.call(d3.axisLeft(newYScale)
          .tickValues(tickValues)
          .tickFormat(d => d.toString()));

        // Update the dots positions
        vizGroup.selectAll('.dot')
          .attr('cx', function(d) {
            const data = d as ProcessedDataPoint;
            return newXScale(data.day);
          })
          .attr('cy', function(d) {
            const data = d as ProcessedDataPoint;
            return newYScale(getDisplayTemp(data.avgTemp));
          });

        // Update the line
        const newLine = d3.line<ProcessedDataPoint>()
          .x(d => newXScale(d.day))
          .y(d => newYScale(getDisplayTemp(d.avgTemp)))
          .curve(d3.curveMonotoneX);

        vizGroup.select('path')
          .datum(processedData)
          .attr('d', newLine);
      });

    d3.select(svg)
      .call(zoom)
      .on('dblclick.zoom', null); // Disable double-click zoom

    // Now create axis groups LAST so they are drawn on top
    const xAxisGroup = g.append('g')
      .attr('class', 'x-axis')
      .attr('transform', `translate(0,${innerHeight})`);

    const yAxisGroup = g.append('g')
      .attr('class', 'y-axis');

    // Initial axis rendering
    xAxisGroup.call(d3.axisBottom(xScale))
      .append('text')
      .attr('x', innerWidth / 2)
      .attr('y', 35)
      .attr('fill', 'currentColor')
      .attr('text-anchor', 'middle')
      .text('Time (Days)');

    yAxisGroup.call(d3.axisLeft(yScale)
      .tickValues(tickValues)
      .tickFormat(d => d.toString()))
      .append('text')
      .attr('transform', 'rotate(-90)')
      .attr('x', -innerHeight / 2)
      .attr('y', -40)
      .attr('fill', 'currentColor')
      .attr('text-anchor', 'middle')
      .text(`Temperature (${getTempUnit()})`);
  }

  function resetZoom() {
    d3.select(svg)
      .transition()
      .duration(750)
      .call(zoom.transform, d3.zoomIdentity);
  }

  onMount(async () => {
    await loadAndProcessData();
  });
</script>

<div class="graph-container">
  <div class="main-content">
    <svg bind:this={svg}></svg>
    <svg bind:this={brushSvg} class="brush-svg"></svg>
  </div>
  <div class="legend">
    <h3>Legend</h3>
    <div class="legend-items">
      <div class="legend-item">
        <div class="legend-color" style="background-color: #1f77b4;"></div>
        <span>Avg Core Body Temp</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background-color: gray; opacity: 0.1;"></div>
        <span>Lights Off</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background-color: yellow; opacity: 0.1;"></div>
        <span>Lights On</span>
      </div>
      {#if dataType === 'female'}
        <div class="legend-item">
          <div class="legend-color estrus-color" style="background-color: #FCCFCE; opacity: 1;"></div>
          <span>Estrus</span>
        </div>
      {/if}
    </div>
    <div class="controls">
      <button class="temp-toggle" on:click={toggleTempUnit}>
        Switch to {useFahrenheit ? 'Celsius' : 'Fahrenheit'}
      </button>
      <select 
        class="data-type-select" 
        bind:value={dataType} 
        on:change={() => updateDataType(dataType)}
      >
        <option value="female">Female Mice</option>
        <option value="male">Male Mice</option>
        <option value="both">All Mice</option>
      </select>
      <button 
        class="estrus-toggle" 
        class:disabled={dataType !== 'female'}
        on:click={() => {
          if (dataType === 'female') {
            showEstrus = !showEstrus;
            isBrushed = false;
            selectedData = [];
            createVisualization();
          }
        }}
      >
        {showEstrus ? 'Hide' : 'Show'} Estrus Period
      </button>
      <button 
        class="reset-brush" 
        class:disabled={!isBrushed}
        on:click={resetBrush}
      >
        Reset Selection
      </button>
    </div>
  </div>
</div>

<style>
  .graph-container {
    width: 100%;
    max-width: 1200px;
    margin: 0 auto;
    position: relative;
    display: flex;
    gap: 20px;
    padding: 20px;
  }

  .main-content {
    flex: 1;
    position: relative;
  }
  
  svg {
    width: 100%;
    height: 100%;
    max-width: 100%;
    max-height: 100%;
  }
  
  .brush-svg {
    cursor: crosshair;
    margin-top: 10px;
  }

  .controls {
    margin-top: 20px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .temp-toggle {
    padding: 8px 12px;
    background-color: #1f77b4;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    width: 100%;
  }

  .temp-toggle:hover {
    background-color: #1668a1;
  }

  .data-type-select {
    padding: 8px 12px;
    background-color: white;
    border: 1px solid #1f77b4;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    color: #1f77b4;
    width: 100%;
  }

  .data-type-select:hover {
    background-color: #f5f5f5;
  }

  .estrus-toggle {
    padding: 8px 12px;
    background-color: #1f77b4;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    width: 100%;
  }

  .estrus-toggle:hover:not(.disabled) {
    background-color: #1668a1;
  }

  .estrus-toggle.disabled {
    background-color: #cccccc;
    cursor: not-allowed;
    opacity: 0.6;
  }

  .reset-brush {
    padding: 8px 12px;
    background-color: #1f77b4;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    width: 100%;
  }

  .reset-brush:hover:not(.disabled) {
    background-color: #1668a1;
  }

  .reset-brush.disabled {
    background-color: #cccccc;
    cursor: not-allowed;
    opacity: 0.6;
  }

  .legend {
    width: 200px;
    padding: 20px;
    background-color: #f8f9fa;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  }

  .legend h3 {
    margin: 0 0 15px 0;
    font-size: 16px;
    color: #333;
  }

  .legend-items {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .legend-item {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .legend-color {
    width: 20px;
    height: 20px;
    border-radius: 4px;
    border: 1px solid #ddd;
  }
  
  /* Style for estrus legend item specifically */
  .estrus-color {
    mix-blend-mode: multiply;
  }

  .legend-item span {
    font-size: 14px;
    color: #555;
  }

  :global(.brush .selection) {
    fill: #1f77b4;
    fill-opacity: 0.1;
    stroke: #1f77b4;
    stroke-width: 1px;
  }
  
  :global(.brush .selection) {
    stroke: #1f77b4;
    fill: #1f77b4;
    fill-opacity: 0.15;
  }
</style>