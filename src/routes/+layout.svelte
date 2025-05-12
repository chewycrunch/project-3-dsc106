<script lang="ts">
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
	{@render children()}
</main>
