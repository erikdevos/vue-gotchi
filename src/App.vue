<template>
		<TamagotchiDevice
			:startGotchi="startGotchi"
			:currentFrame="currentFrame"
			:spritePath="spritePath"
			:spriteDirection="neutral"
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
		currentState: 'idle',
		currentFrame: 'sprite_calm.png',
		spritePath: 'sprites/sprite_',
		currentSpriteImagePath: 'fout',
		spriteDirection: 'neutral'
	};
},
	
	computed: {

	},
	
	methods: {
		idle() {
		console.log('idle');

		const generateFrameName = (name) => `sprite_${name}.png`;
		const animationFrameNames = ['calm', 'casual', 'calm', 'casual', 'neutral']; // Add more names as needed

		const numFrames = animationFrameNames.length;
		const animationFrames = animationFrameNames.map(generateFrameName);

		let index = 0;

		const animate = () => {
			this.currentFrame = animationFrames[index];

			// Increment the index or reset it to 0 if it reaches the end of the array
			index = (index + 1) % numFrames;

			// Set a timeout for the next frame change (adjust the delay as needed)
			setTimeout(animate, 1000); // 1000 milliseconds = 1 second
		};
		

		// Start the animation
		animate();
		},
	},

	created() {
		// Always start with idle animation
		this.idle();
	}
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
