<template>
		<TamagotchiDevice
			:startGotchi="startGotchi"
			:spritePath="spritePath"
			:spriteDirection="neutral"
			:gotchiEnergy="gotchiEnergy"
			:gotchiHappyness="gotchiHappyness"
			:gotchiStatus="gotchiStatus"
			@gotchi-status-update="updateGotchiStatus"
		/>
</template>


<script>
import TamagotchiDevice from './components/TamagotchiDevice.vue';

export default {
	name: 'App',
	components: {
		TamagotchiDevice
	},

	data() {
		return {
			startGotchi: true,
			spritePath: 'sprites/sprite_',
			spriteDirection: 'neutral',
			gotchiEnergy: 100,
			gotchiHappyness: 100,
			gotchiStatus: 'alive'
		};
	},
	
	methods: {
		updateGotchiStatus(status) {
			this.gotchiStatus = status;
		},
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
