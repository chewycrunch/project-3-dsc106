<script lang="ts">
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  // Component props
  export let width = 800;
  export let height = 400;
  export let margin = { top: 40, right: 60, bottom: 40, left: 60 };
  export let brushHeight = 60; // Height for the brush component

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
  let brush: d3.BrushBehavior<unknown>;
  let currentBrushSelection: [number, number] | null = null;

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

  // Update data type and reload
  function updateDataType(newType: DataType) {
    dataType = newType;
    // Reset estrus toggle when switching to male or both
    if (newType !== 'female') {
      showEstrus = false;
    }
    loadAndProcessData();
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
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    // Create a group for the visualization
    const vizGroup = g.append('g')
      .attr('class', 'visualization');

    // Create background group for light/dark periods
    const bgGroup = vizGroup.append('g')
      .attr('class', 'background');

    // Add light/dark period shading
    for (let day = 0; day < 14; day++) {
      // Dark period (gray)
      bgGroup.append('rect')
        .attr('x', xScale(Math.max(day, xDomain[0])))
        .attr('y', 0)
        .attr('width', xScale(Math.min(day + 0.5, xDomain[1])) - xScale(Math.max(day, xDomain[0])))
        .attr('height', innerHeight)
        .attr('fill', '#F2F2F2')
        .attr('opacity', 1);
				
      // Light period (yellow)
      bgGroup.append('rect')
        .attr('x', xScale(Math.max(day + 0.5, xDomain[0])))
        .attr('y', 0)
        .attr('width', xScale(Math.min(day + 1, xDomain[1])) - xScale(Math.max(day + 0.5, xDomain[0])))
        .attr('height', innerHeight)
        .attr('fill', '#FFFFEC')
        .attr('opacity', 1);
    }

    // Add estrus day highlighting only if enabled and female mice selected
    if (showEstrus && dataType === 'female') {
      const estrusGroup = vizGroup.append('g')
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
      .datum(visibleData)
      .attr('fill', 'none')
      .attr('stroke', '#1f77b4')
      .attr('stroke-width', 1.5)
      .attr('d', line);

    // Add title (outside of vizGroup)
    const titleText = dataType === 'female' 
      ? 'Average Core Body Temperature (Female Mice)' 
      : dataType === 'male' 
        ? 'Average Core Body Temperature (Male Mice)'
        : 'Average Core Body Temperature (All Mice)';
        
    g.append('text')
      .attr('class', 'title')
      .attr('x', innerWidth / 2)
      .attr('y', -10)
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

    // Add interactive dots for visible data
    vizGroup.selectAll('.dot')
      .data(visibleData)
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

    // Create axis groups
    const xAxisGroup = g.append('g')
      .attr('class', 'x-axis')
      .attr('transform', `translate(0,${innerHeight})`);

    const yAxisGroup = g.append('g')
      .attr('class', 'y-axis');

    // Initial axis rendering
    xAxisGroup.call(formatXAxis(xScale))
      .append('text')
      .attr('x', innerWidth / 2)
      .attr('y', 35)
      .attr('fill', 'currentColor')
      .attr('text-anchor', 'middle')
      .text('Time');

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

  function createBrushVisualization() {
    if (!processedData.length) return;

    // Clear previous brush visualization
    d3.select(brushSvg).selectAll("*").remove();

    const innerWidth = width - margin.left - margin.right;
    const brushMargin = { top: 5, right: margin.right, bottom: 20, left: margin.left };
    const innerHeight = brushHeight - brushMargin.top - brushMargin.bottom;

    // Create scales
    const xScale = d3.scaleLinear()
      .domain([0, 14]) // Full 14 days
      .range([0, innerWidth]);

    const yScale = d3.scaleLinear()
      .domain(getYDomain())
      .range([innerHeight, 0]);

    // Create SVG group
    const g = d3.select(brushSvg)
      .attr('width', width)
      .attr('height', brushHeight)
      .append('g')
      .attr('transform', `translate(${brushMargin.left},${brushMargin.top})`);

    // Add background for context
    for (let day = 0; day < 14; day++) {
      // Dark period (gray)
      g.append('rect')
        .attr('x', xScale(day))
        .attr('y', 0)
        .attr('width', xScale(day + 0.5) - xScale(day))
        .attr('height', innerHeight)
        .attr('fill', '#F2F2F2')
        .attr('opacity', 1);

      // Light period (yellow)
      g.append('rect')
        .attr('x', xScale(day + 0.5))
        .attr('y', 0)
        .attr('width', xScale(day + 1) - xScale(day + 0.5))
        .attr('height', innerHeight)
        .attr('fill', '#FFFFEC')
        .attr('opacity', 1);
    }
    
    // Add estrus day highlighting to brush if needed
    if (showEstrus && dataType === 'female') {
      [2, 6, 10].forEach(day => {
        g.append('rect')
          .attr('x', xScale(day))
          .attr('y', 0)
          .attr('width', xScale(day + 1) - xScale(day))
          .attr('height', innerHeight)
          .attr('fill', '#FCCFCE')
          .attr('opacity', 1)
          .style('mix-blend-mode', 'multiply');
      });
    }

    // Create line generator
    const line = d3.line<ProcessedDataPoint>()
      .x(d => xScale(d.day))
      .y(d => yScale(getDisplayTemp(d.avgTemp)))
      .curve(d3.curveMonotoneX);

    // Add the temperature line
    g.append('path')
      .datum(processedData)
      .attr('fill', 'none')
      .attr('stroke', '#1f77b4')
      .attr('stroke-width', 1)
      .attr('d', line);

    // Create brush
    brush = d3.brushX()
      .extent([[0, 0], [innerWidth, innerHeight]])
      .on('end', brushed);

    // Add the brush
    const brushG = g.append('g')
      .attr('class', 'brush')
      .call(brush);

    // If we have a previous selection, apply it
    if (currentBrushSelection) {
      brushG.call(brush.move, [
        xScale(currentBrushSelection[0]), 
        xScale(currentBrushSelection[1])
      ]);
    }

    // Add x-axis
    g.append('g')
      .attr('class', 'x-axis')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(d3.axisBottom(xScale).ticks(7).tickFormat(d => `${d}`));

    function brushed(event: d3.D3BrushEvent<unknown>) {
      if (!event.selection) {
        // If the brush is cleared, reset to full view
        currentBrushSelection = null;
        createVisualization();
        return;
      }

      // Convert the pixel selection to data values
      const [x0, x1] = event.selection as [number, number];
      const newDomain: [number, number] = [
        xScale.invert(x0), 
        xScale.invert(x1)
      ];
      
      // Save current selection
      currentBrushSelection = newDomain;
      
      // Update main visualization with new domain
      createVisualization();
    }
  }

  function resetView() {
    // Clear brush selection
    currentBrushSelection = null;
    
    // Recreate both visualizations
    createBrushVisualization();
    createVisualization();
  }

  onMount(async () => {
    await loadAndProcessData();
    
    // Remove any lingering tooltips from previous renders
    d3.selectAll('.mouse-tooltip').remove();
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
        <div class="legend-color" style="background-color: #F2F2F2; opacity: 1;"></div>
        <span>Lights Off</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background-color: #FFFFEC; opacity: 1;"></div>
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
            createVisualization();
            createBrushVisualization();
          }
        }}
      >
        {showEstrus ? 'Hide' : 'Show'} Estrus Period
      </button>
      <button class="reset-zoom" on:click={resetView}>
        Reset View
      </button>
    </div>
  </div>
</div>

<style>
  .graph-container {
    width: 100%;
    max-width: 1000px;
    margin: 0 auto;
    position: relative;
    display: flex;
    gap: 20px;
  }

  .main-content {
    flex: 1;
    position: relative;
    display: flex;
    flex-direction: column;
  }
  
  svg {
    width: 100%;
    height: auto;
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

  .reset-zoom {
    padding: 8px 12px;
    background-color: #1f77b4;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    width: 100%;
    margin-top: 10px;
  }

  .reset-zoom:hover {
    background-color: #1668a1;
  }
  
  :global(.brush .selection) {
    stroke: #1f77b4;
    fill: #1f77b4;
    fill-opacity: 0.15;
  }
</style>