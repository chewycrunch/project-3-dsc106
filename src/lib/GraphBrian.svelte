<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as d3 from 'd3';

  // Component props
  const aspectRatio = 0.6; // Height will be 60% of width
  export let width = Math.min(window.innerWidth * 0.75, 1700);
  export let height = width * aspectRatio; // Calculate height based on width
  export let margin = { 
    top: 60,    // Increased for larger title
    right: 80,  // Increased to prevent overlap with legend
    bottom: 80, // Increased for larger axis labels
    left: 80    // Increased for larger axis labels
  };

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
  let data: any[] = [];
  let processedData: ProcessedDataPoint[] = [];
  let useFahrenheit = false;
  let dataType: DataType = 'female';

  let resizeObserver: ResizeObserver;
  let svgContainer: HTMLDivElement;

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

    // Update visualization with new data
    createVisualization();
  }

  // Update data type and reload
  function updateDataType(newType: DataType) {
    dataType = newType;
    loadAndProcessData();
  }

  function createVisualization() {
    if (!processedData.length) return;

    // Clear previous visualization
    d3.select(svg).selectAll("*").remove();

    const innerWidth = width - margin.left - margin.right;
    const innerHeight = height - margin.top - margin.bottom;

    // Create scales
    const xScale = d3.scaleLinear()
      .domain([0, 14]) // 14 days
      .range([0, innerWidth]);

    const yScale = d3.scaleLinear()
      .domain(getYDomain())
      .range([innerHeight, 0]);

    // Create SVG group
    const g = d3.select(svg)
      .attr('width', width)
      .attr('height', height)
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    // Add light/dark period shading
    for (let day = 0; day < 14; day++) {
      // Dark period (gray)
      g.append('rect')
        .attr('x', xScale(day))
        .attr('y', 0)
        .attr('width', xScale(day + 0.5) - xScale(day))
        .attr('height', innerHeight)
        .attr('fill', 'gray')
        .attr('opacity', 0.1);

      // Light period (yellow)
      g.append('rect')
        .attr('x', xScale(day + 0.5))
        .attr('y', 0)
        .attr('width', xScale(day + 1) - xScale(day + 0.5))
        .attr('height', innerHeight)
        .attr('fill', 'yellow')
        .attr('opacity', 0.1);
    }

    // Add estrus day highlighting
    [2, 6, 10].forEach(day => {
      g.append('rect')
        .attr('x', xScale(day))
        .attr('y', 0)
        .attr('width', xScale(day + 1) - xScale(day))
        .attr('height', innerHeight)
        .attr('fill', 'red')
        .attr('opacity', 0.2);
    });

    // Create line generator with proper typing
    const line = d3.line<ProcessedDataPoint>()
      .x(d => xScale(d.day))
      .y(d => yScale(getDisplayTemp(d.avgTemp)))
      .curve(d3.curveMonotoneX);

    // Add the temperature line
    g.append('path')
      .datum(processedData)
      .attr('fill', 'none')
      .attr('stroke', '#1f77b4')
      .attr('stroke-width', 1.5)
      .attr('d', line);

    // Add axes
    const xAxis = d3.axisBottom(xScale)
      .tickSize(6)
      .tickPadding(8);
    
    // Create y-axis with appropriate ticks
    let yAxis;
    const domain = getYDomain();
    const tickCount = 6; // Number of ticks including boundaries
    const step = (domain[1] - domain[0]) / (tickCount - 1);
    const tickValues = Array.from({ length: tickCount }, (_, i) => 
      +(domain[0] + (step * i)).toFixed(1)
    );
    
    yAxis = d3.axisLeft(yScale)
      .tickValues(tickValues)
      .tickFormat(d => d.toString());

    g.append('g')
      .attr('transform', `translate(0,${innerHeight})`)
      .call(xAxis)
      .append('text')
      .attr('x', innerWidth / 2)
      .attr('y', 45)
      .attr('fill', 'currentColor')
      .attr('text-anchor', 'middle')
      .style('font-size', '18px')
      .text('Time (Days)');

    g.append('g')
      .call(yAxis)
      .append('text')
      .attr('transform', 'rotate(-90)')
      .attr('x', -innerHeight / 2)
      .attr('y', -50)
      .attr('fill', 'currentColor')
      .attr('text-anchor', 'middle')
      .style('font-size', '18px')
      .text(`Temperature (${getTempUnit()})`);

    // Style the axis ticks
    g.selectAll('.tick text')
      .style('font-size', '14px');

    // Add title with larger font
    g.append('text')
      .attr('x', innerWidth / 2)
      .attr('y', -20)
      .attr('text-anchor', 'middle')
      .style('font-size', '24px')
      .style('font-weight', 'bold')
      .text('Average Core Body Temperature Over 14 Days (Female Mice)');

    // Add tooltip
    const tooltip = d3.select('body')
      .append('div')
      .style('position', 'absolute')
      .style('background-color', 'white')
      .style('padding', '10px')
      .style('border', '1px solid #ddd')
      .style('border-radius', '4px')
      .style('pointer-events', 'none')
      .style('opacity', 0)
      .style('font-size', '14px')
      .style('box-shadow', '0 2px 4px rgba(0,0,0,0.1)');

    // Add interactive dots
    g.selectAll('.dot')
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
           Period: ${!d.light ? 'Lights On' : 'Lights Off'}<br/>
           Estrus: ${d.estrus ? 'Yes' : 'No'}`
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
  }

  // Handle resize
  function handleResize(entries: ResizeObserverEntry[]) {
    for (const entry of entries) {
      if (svg) {
        // Update width based on viewport
        width = Math.min(window.innerWidth * 0.75, 1700);
        // Update height based on width ratio
        height = width * aspectRatio;
        createVisualization();
      }
    }
  }

  onMount(async () => {
    await loadAndProcessData();
    
    // Set up resize observer
    resizeObserver = new ResizeObserver(handleResize);
    if (svgContainer) {
      resizeObserver.observe(svgContainer);
    }
  });

  onDestroy(() => {
    if (resizeObserver) {
      resizeObserver.disconnect();
    }
  });
</script>

<div class="graph-container">
  <div class="main-content">
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
    </div>
    <div class="svg-container" bind:this={svgContainer}>
      <svg bind:this={svg}></svg>
    </div>
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
      <div class="legend-item">
        <div class="legend-color" style="background-color: red; opacity: 0.2;"></div>
        <span>Estrus</span>
      </div>
    </div>
  </div>
</div>

<style>
  .graph-container {
    width: 75vw;
    max-width: 1700px;
    margin: 0 auto;
    position: relative;
    display: flex;
    gap: 40px;
    padding: 20px;
    align-items: center;
    justify-content: center;
  }

  .main-content {
    flex: 0 1 auto;
    position: relative;
    display: flex;
    flex-direction: column;
    min-width: 0;
    aspect-ratio: 1/0.6; /* Match the width/height ratio */
  }

  .svg-container {
    width: 100%;
    height: 100%;
    position: relative;
    overflow: visible;
    display: flex;
    align-items: center;
  }

  svg {
    width: 100%;
    height: 100%;
    overflow: visible;
  }

  .controls {
    position: absolute;
    top: 20px;
    right: 20px;
    display: flex;
    gap: 12px;
    z-index: 1;
  }

  .temp-toggle {
    padding: 8px 16px;
    background-color: #1f77b4;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 500;
  }

  .temp-toggle:hover {
    background-color: #1668a1;
  }

  .data-type-select {
    padding: 8px 16px;
    background-color: white;
    border: 1px solid #1f77b4;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    color: #1f77b4;
    font-weight: 500;
  }

  .data-type-select:hover {
    background-color: #f5f5f5;
  }

  .legend {
    flex: 0 0 240px;
    height: 100%;
    aspect-ratio: 1/0.6; /* Match the graph's aspect ratio */
    padding: 25px;
    background-color: #f8f9fa;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    display: flex;
    flex-direction: column;
    justify-content: center;
  }

  .legend h3 {
    margin: 0 0 20px 0;
    font-size: 20px;
    color: #333;
    font-weight: bold;
    text-align: center; /* Center the legend title */
  }

  .legend-items {
    display: flex;
    flex-direction: column;
    gap: 16px;
    justify-content: center; /* Center items vertically */
    flex: 1; /* Take up remaining space */
  }

  .legend-item {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .legend-color {
    width: 24px;
    height: 24px;
    border-radius: 4px;
    border: 1px solid #ddd;
  }

  .legend-item span {
    font-size: 16px;
    color: #555;
  }

  @media (max-width: 768px) {
    .graph-container {
      width: 95vw;
      flex-direction: column;
      gap: 20px;
      padding: 10px;
      align-items: stretch;
    }

    .main-content {
      width: 100%;
      aspect-ratio: 1/0.6; /* Maintain aspect ratio on mobile */
    }

    .legend {
      width: 100%;
      height: auto;
      aspect-ratio: auto; /* Allow flexible height on mobile */
      position: relative;
      top: 0;
    }

    .legend-items {
      justify-content: flex-start; /* Align items to top on mobile */
    }
  }
</style>