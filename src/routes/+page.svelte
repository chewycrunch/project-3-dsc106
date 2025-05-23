<script lang="ts">
	import Graph from '$lib/Graph.svelte';
</script>

<!-- Title -->
<div class="mt-20 flex flex-col gap-x-2.5">
	<h1>Mouse Data - Pulse of the Night</h1>
	<h2>
		How does the thermoregulation of mice differ with changes to gender, time of day, and estrus
		period?
	</h2>
	<h2>
		Dynamic visualization of mouse behavior, thermoregulation, and red-band estrus under 12h
		light/dark.
	</h2>
	<h3>Brian Can, David Chew, Derek Kuang, Aathi Muthu</h3>
	<span class="text-2xl font-bold text-red-500"
		>DISCLAIMER: Data takes about 15 secs to load. Chart will not be visible until then.</span
	>
</div>

<p class="my-6 text-justify leading-relaxed">
	Our visualization centers on three coordinated components that let users explore mouse activity,
	temperature, and estrus cycles with minimal friction. A light/dark color scheme immediately
	communicates the 12-hour on/off rhythm: light hues for "lights on" and dark shades for "lights
	off." Estrus days are highlighted in red only when viewing female subjects (the estrus toggle is
	hidden for males and disabled when "Both" sexes are selected). Users can switch body-temperature
	units between Fahrenheit and Celsius. Users can also use the time slider at the bottom to zoom in
	on a specific time period of their interest to better examine detailed fluctuations in mouse
	activity, temperature variations, and estrus timing within the broader circadian cycle.
</p>

<Graph />

<p class="my-6 text-justify leading-relaxed">
	We chose solid light/dark bands to make the 12-hour on/off cycle immediately legible (a blue-black
	gradient proved too subtle), overlaid semi-opaque red rectangles for estrus so users can instantly
	see how activity and temperature shift during those periods (and hide that control for males or
	“Both” to avoid dead-end toggles), and an explicit °F/°C switch that not only relabels the axis
	but dynamically rescales tick intervals. Free-form pan/zoom lets users dive into any anomaly,
	while the draggable range selector provides quick “overview + detail” highlighting of min/max
	values. An alternative to this approach was to directly pan and zoom on the graph. However this
	caused issues of zooming into the wrong spot since we would manually be zooming with our mouse.
	Together, these encodings and interactions balance clarity and flexibility, letting users fluidly
	explore circadian rhythms without visual overload.
</p>

<p class="my-6 text-justify leading-relaxed">
	Our two-week development is included in a single GitHub repository. We started off with a group
	brainstorming session, then distributed tasks among the four of us: Brian implemented the D3 map
	of light/dark intervals, created the temperature-unit toggle and axis rescaling, Aathi built the
	gender and estrus filters, created the time-range selector and pan/zoom behavior. David and Derek
	fixed the styling and added the write up for the whole project. In total, we logged roughly 16
	person-hours (about four each). Integrating the zoom and drag behaviors in D3 proved the biggest
	time sink—it took nearly two hours to nail smooth interactions without layout jank. Once those
	were stable, integrating the remaining controls and polishing the color scheme fell into place
	quickly.
</p>

<style scoped>
	@reference '$lib/../app.css';
	h1 {
		@apply font-serif text-4xl font-bold tracking-wide;
	}

	h2 {
		@apply font-serif text-2xl font-medium tracking-wide;
	}

	h3 {
		@apply font-serif text-lg font-normal;
	}

	p {
		@apply font-sans text-base font-normal;
	}
</style>
