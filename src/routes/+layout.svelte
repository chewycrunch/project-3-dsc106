<script lang="ts">
	import Container from '$lib/Container.svelte';
	import Header from '$lib/Header.svelte';
	import '../app.css';
	import '../scroll.css';

	let { children } = $props();
</script>

<svelte:head>
	<title>Mouse</title>
	<meta name="description" content="View your favorite basketball content." />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
</svelte:head>

<Header />
<main>
	<Container>
		<div class="flex flex-col gap-x-2.5">
			<h1 class="title">Mouse Data; A Better Title Here</h1>
			<h2>A cool insight of TBD</h2>
			<h3 class="subtitle">Brian Can, David Chew, Derek Kuang, Aathi Muthu</h3>
		</div>

		<p>temp writeup here</p>

		{@render children()}

		<p>temp writeup here</p>
	</Container>
</main>

<style scoped>
	@reference "$lib/../app.css";
	.title {
		@apply font-serif text-4xl font-bold;
	}
</style>
