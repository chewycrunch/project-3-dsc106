<script lang="ts">
	import Container from '$lib/Container.svelte';
	import '../app.css';
	import '../scroll.css';

	let { children } = $props();
</script>

<svelte:head>
	<title>Mouse</title>
	<meta name="description" content="View your favorite basketball content." />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
</svelte:head>

<main>
	<Container>
		{@render children()}
	</Container>
</main>
