<template>
	<div class="container">
		<div class="gotchi-frame">
			<div class="frame-screen">
				<!-- Status bars -->
				<div class="status-bars-wrapper">
					<div class="status-bar happiness-bar">
						<div class="bar-value happiness-bar-value" :style="barStyle(happiness)"></div>
						<span class="bar-label">happy</span>
					</div>
					<div class="status-bar energy-bar">
						<div class="bar-value energy-bar-value" :style="barStyle(energy)"></div>
						<span class="bar-label">energy</span>
					</div>
				</div>

				<!-- Sprite -->
				<div class="frame-wrapper" :class="{ 'is-acting': isActing }">
					<img alt="" :src="currentSprite">
				</div>
			</div>

			<!-- Buttons -->
			<div class="button-bar">
				<button @click="doEat" :disabled="!canAct" title="Feed your gotchi (+energy)">
					Eat
				</button>
				<button @click="doPlay" :disabled="!canAct" title="Play with your gotchi (+happy, -energy)">
					Play
				</button>
				<button @click="doSleep" :disabled="isDead || isActing" title="Put your gotchi to sleep">
					{{ isSleeping ? 'Wake' : 'Sleep' }}
				</button>
			</div>
		</div>
	</div>
</template>

<script>
// ─── Animation Config ────────────────────────────────────────────────────────
// Edit frame sequences and timing here without touching game logic.
// Frame names map to /assets/sprites/sprite_<name>.png
const ANIMATIONS = {
	idle:     { frames: ['calm', 'casual', 'neutral', 'calm'],                        interval: 900,  loop: true  },
	eating:   { frames: ['active', 'happy', 'joy', 'happy', 'neutral'],               interval: 380,  loop: false },
	playing:  { frames: ['fun', 'kiss_01', 'kiss_02', 'happy', 'joy', 'neutral'],     interval: 380,  loop: false },
	sleeping: { frames: ['sleep'],                                                     interval: 2000, loop: true  },
	waking:   { frames: ['neutral', 'casual', 'calm'],                                interval: 600,  loop: false },
	dead:     { frames: ['shocked', 'sad'],                                           interval: 1400, loop: true  },
};

// ─── Game Constants ───────────────────────────────────────────────────────────
const DECAY_INTERVAL  = 6000; // ms per game tick
const HAPPINESS_DECAY = 3;    // happiness lost per tick
const ENERGY_DECAY    = 1.5;  // energy lost per tick
const SLEEP_REGEN     = 8;    // energy gained per tick while sleeping

export default {
	name: 'TamagotchiDevice',

	data() {
		return {
			happiness:   80,
			energy:      80,
			status:      'idle', // idle | eating | playing | sleeping | waking | dead
			currentFrame: 'calm',
			isActing:    false,  // true only during one-shot action animations
			animTimeout: null,
			decayInterval: null,
		};
	},

	computed: {
		currentSprite() {
			return require(`@/assets/sprites/sprite_${this.currentFrame}.png`);
		},
		isSleeping() {
			return this.status === 'sleeping';
		},
		isDead() {
			return this.status === 'dead';
		},
		canAct() {
			return !this.isDead && !this.isSleeping && !this.isActing;
		},
	},

	methods: {
		// ── Visuals ──────────────────────────────────────────────────────────
		barStyle(value) {
			const v = Math.max(0, Math.min(100, value));
			let bg = '#53bf8e';
			if (v <= 30) bg = '#e74c3c';
			else if (v <= 55) bg = '#f39c12';
			return { width: v + '%', backgroundColor: bg };
		},

		// ── Animation engine ─────────────────────────────────────────────────
		playAnimation(animKey, onDone) {
			this.stopAnimation();
			const anim = ANIMATIONS[animKey];
			if (!anim) return;

			this.isActing = !anim.loop;
			let index = 0;

			const tick = () => {
				this.currentFrame = anim.frames[index];
				index++;

				if (index < anim.frames.length) {
					this.animTimeout = setTimeout(tick, anim.interval);
				} else if (anim.loop) {
					index = 0;
					this.animTimeout = setTimeout(tick, anim.interval);
				} else {
					this.isActing = false;
					if (onDone) onDone();
				}
			};

			tick();
		},

		stopAnimation() {
			if (this.animTimeout) {
				clearTimeout(this.animTimeout);
				this.animTimeout = null;
			}
			this.isActing = false;
		},

		startIdle() {
			this.status = 'idle';
			this.playAnimation('idle');
		},

		// ── Actions ───────────────────────────────────────────────────────────
		doEat() {
			if (!this.canAct) return;
			this.energy    = Math.min(100, this.energy + 25);
			this.happiness = Math.min(100, this.happiness + 5);
			this.status = 'eating';
			this.playAnimation('eating', () => this.startIdle());
		},

		doPlay() {
			if (!this.canAct) return;
			this.happiness = Math.min(100, this.happiness + 20);
			this.energy    = Math.max(0, this.energy - 15);
			this.status = 'playing';
			this.playAnimation('playing', () => this.startIdle());
		},

		doSleep() {
			if (this.isDead || this.isActing) return;
			if (this.isSleeping) {
				this.stopAnimation();
				this.happiness = Math.min(100, this.happiness + 10);
				this.status = 'waking';
				this.playAnimation('waking', () => this.startIdle());
			} else {
				this.stopAnimation();
				this.status = 'sleeping';
				this.playAnimation('sleeping');
			}
		},

		// ── Game loop ─────────────────────────────────────────────────────────
		die() {
			this.stopAnimation();
			this.status = 'dead';
			this.playAnimation('dead');
			clearInterval(this.decayInterval);
		},

		startDecay() {
			this.decayInterval = setInterval(() => {
				if (this.isDead) return;

				if (this.isSleeping) {
					this.energy = Math.min(100, this.energy + SLEEP_REGEN);
				} else {
					this.happiness = Math.max(0, this.happiness - HAPPINESS_DECAY);
					this.energy    = Math.max(0, this.energy - ENERGY_DECAY);
				}

				if (this.happiness <= 0 || this.energy <= 0) {
					this.die();
				}
			}, DECAY_INTERVAL);
		},
	},

	created() {
		this.startIdle();
		this.startDecay();
	},

	beforeUnmount() {
		this.stopAnimation();
		clearInterval(this.decayInterval);
	},
};
</script>

<style scoped>

.container {
	margin-left: auto;
	margin-right: auto;
	height: 100%;
	width: 100%;
	max-width: 600px;
	display: flex;
	align-items: center;
}

.gotchi-frame {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	width: 100%;
	height: 80vh;
	background-image: url("../assets/egg-shape.svg");
	background-position: center center;
	background-repeat: no-repeat;
	background-size: contain;

	.frame-screen {
		position: relative;
		background-color: #c8d8a0;
		border: solid 8px black;
		border-radius: 1rem;
		height: 18rem;
		width: 15rem;
		display: flex;
		justify-content: center;
		margin-top: 5rem;
		margin-bottom: 3rem;
		box-shadow: inset 0 0 12px rgba(0,0,0,0.25);
	}

	.status-bars-wrapper {
		position: absolute;
		width: calc(100% - 1rem);
		margin-top: 0.5rem;
		margin-left: 0.5rem;
		margin-right: 0.5rem;
		gap: 0.5rem;
		display: flex;
	}

	.status-bar {
		display: flex;
		align-items: center;
		justify-content: center;
		height: 1rem;
		border: solid 3px #000000;
		border-radius: 0.3rem;
		font-weight: bold;
		font-size: 0.8rem;
		flex: 1;
		text-align: center;
		position: relative;
		overflow: hidden;

		.bar-value {
			height: 100%;
			position: absolute;
			left: 0;
			top: 0;
			z-index: 2;
			transition: width 0.4s ease, background-color 0.4s ease;
		}

		.bar-label {
			z-index: 3;
			position: relative;
		}
	}

	.frame-wrapper {
		width: 80%;
		align-self: end;
		margin-bottom: 1rem;

		&.is-acting img {
			animation: sprite-bounce 0.35s ease infinite alternate;
		}

		> img {
			width: 100%;
			height: auto;
			image-rendering: crisp-edges;
			filter: grayscale(1);
			object-fit: contain;
		}
	}
}

.button-bar {
	display: flex;
	gap: 1rem;

	button {
		all: unset;
		display: inline-flex;
		align-items: center;
		justify-content: center;
		padding: 1.2rem;
		border-radius: 100%;
		background-color: #FFFFFF;
		aspect-ratio: 1/1;
		text-align: center;
		font-family: "Comic Sans MS", "Comic Sans", cursive;
		font-size: 1rem;
		font-weight: 900;
		border: solid 5px #000000;
		cursor: pointer;
		user-select: none;
		box-shadow: rgba(0, 0, 0, 0.5) 1px 1px 5px;
		transition: background-color 0.15s ease;

		&:hover:not(:disabled) {
			background-color: salmon;
			color: #000000;
			transform: translateY(0.1rem);
			box-shadow: rgba(0, 0, 0, 0.5) 1px 1px 3px;
		}

		&:active:not(:disabled) {
			transform: translateY(0.3rem);
			box-shadow: rgba(0, 0, 0, 0.3) 1px 1px 0px;
		}

		&:disabled {
			opacity: 0.4;
			cursor: not-allowed;
		}
	}
}

@keyframes sprite-bounce {
	from { transform: translateY(0); }
	to   { transform: translateY(-6px); }
}

</style>
