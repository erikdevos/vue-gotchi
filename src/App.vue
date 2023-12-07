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
    spriteImagePath() {
      console.log(`@assets${this.spritePath}${this.currentFrame}`);
      return `@assets${this.spritePath}${this.currentFrame}`;
    },
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
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
