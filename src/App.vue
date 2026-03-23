<template>
	<TamagotchiDevice />
</template>

<script>
import TamagotchiDevice from './components/TamagotchiDevice.vue';

export default {
	name: 'App',
	components: {
		TamagotchiDevice
	},
}
</script>

<style>

body {
	margin: 0;
	display: flex;
	align-items: center;
	justify-content: center;
}

#app {
	font-family: Avenir, Helvetica, Arial, sans-serif;
	text-align: center;
	color: #2c3e50;
	margin: 0;
	height: 100vh;
	width: 100%;
	background-image: url('/src/assets/gotchi-background.webp');
	background-size: cover;
	background-repeat: no-repeat;
	background-position: center center;
	display: flex;
	align-items: center;
}
</style>
