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
				<button @click="doPet" :disabled="!canAct" title="Pet your gotchi (+happiness)">
					Pet
				</button>
				<button v-if="isDead" @click="restart" class="restart-btn" title="Start a new gotchi">
					Restart
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
	idle_happy: { frames: ['happy', 'joy', 'happy', 'calm'],                         interval: 900,  loop: true  },
	idle_sad:   { frames: ['sad', 'neutral', 'sad', 'calm'],                         interval: 900,  loop: true  },
	idle_mad:   { frames: ['mad', 'sad', 'mad', 'neutral'],                          interval: 900,  loop: true  },
	eating:   { frames: ['active', 'happy', 'joy', 'happy', 'neutral'],               interval: 380,  loop: false },
	playing:  { frames: ['fun', 'kiss_01', 'kiss_02', 'happy', 'joy', 'neutral'],     interval: 380,  loop: false },
	petting:  { frames: ['kiss_01', 'kiss_02', 'happy', 'joy', 'neutral'],           interval: 400,  loop: false },
	sleeping: { frames: ['sleep'],                                                     interval: 2000, loop: true  },
	waking:   { frames: ['neutral', 'casual', 'calm'],                                interval: 600,  loop: false },
	sick:     { frames: ['mad', 'sad', 'mad'],                                         interval: 800,  loop: true  },
	overfeeding: { frames: ['mad', 'sad', 'mad', 'sad'],                               interval: 400,  loop: false },
	attention: { frames: ['mad', 'sad', 'mad', 'neutral'],                            interval: 600,  loop: true  },
	dead:     { frames: ['shocked', 'sad'],                                           interval: 1400, loop: true  },
};

// ─── Game Constants ───────────────────────────────────────────────────────────
const DECAY_INTERVAL  = 6000; // ms per game tick
const HAPPINESS_DECAY = 3;    // base happiness lost per tick
const ENERGY_DECAY    = 1.5;  // energy lost per tick
const SLEEP_REGEN     = 8;    // energy gained per tick while sleeping
const ENERGY_CRISIS_THRESHOLD = 25; // below this, happiness decays faster
const SICKNESS_THRESHOLD = 15; // below this energy, gotchi gets sick
const ATTENTION_THRESHOLD = 35; // below this happiness, gotchi seeks attention
const MOOD_INERTIA = 0.1; // how quickly mood changes (0-1)
const OVERFEED_BASE_PENALTY = 5; // base happiness lost when overfed
const OVERFEED_SICKNESS_THRESHOLD = 4; // overfeed count to trigger sickness
const OVERPET_THRESHOLD = 8; // pet count to trigger overpetting penalty
const OVERPET_HAPPINESS_PENALTY = 3; // happiness lost when overpetted

export default {
	name: 'TamagotchiDevice',

	data() {
		return {
			happiness:   80,
			energy:      80,
			mood:        80, // 0-100, changes slowly based on happiness
			status:      'idle', // idle | eating | playing | sleeping | waking | sick | attention | dead
			currentFrame: 'calm',
			isActing:    false,  // true only during one-shot action animations
			animTimeout: null,
			decayInterval: null,
			attentionInterval: null, // for attention-seeking behavior
			petCount: 0, // track pet usage for diminishing returns
			overfeedCount: 0, // track overfeeding for sickness/death
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
			return !this.isDead && !this.isSleeping && !this.isActing && this.status !== 'attention';
		},

		moodState() {
			if (this.mood >= 80) return 'happy';
			if (this.mood >= 50) return 'neutral';
			if (this.mood >= 25) return 'sad';
			return 'mad';
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
			// Choose idle animation based on mood
			let animKey = 'idle';
			if (this.mood >= 80) animKey = 'idle_happy';
			else if (this.mood >= 50) animKey = 'idle';
			else if (this.mood >= 25) animKey = 'idle_sad';
			else animKey = 'idle_mad';

			this.status = 'idle';
			this.playAnimation(animKey);
		},

		// ── Actions ───────────────────────────────────────────────────────────
		doEat() {
			if (!this.canAct) return;

			// Check for overfeeding
			if (this.energy >= 100) {
				this.overfeedCount++;
				
				// Escalating penalty: each overfeed hurts more
				const penalty = OVERFEED_BASE_PENALTY + (this.overfeedCount - 1) * 2;
				this.happiness = Math.max(0, this.happiness - penalty);

				// Check sickness threshold
				if (this.overfeedCount >= OVERFEED_SICKNESS_THRESHOLD) {
					this.startSickness();
					return;
				}

				// Show overfeeding animation
				this.status = 'overfeeding';
				this.playAnimation('overfeeding', () => this.startIdle());
			} else {
				this.energy = Math.min(100, this.energy + 25);
				this.happiness = Math.min(100, this.happiness + 5);
				this.status = 'eating';
				this.playAnimation('eating', () => this.startIdle());
			}
		},

		doPlay() {
			if (!this.canAct) return;
			this.happiness = Math.min(100, this.happiness + 20);
			this.energy    = Math.max(0, this.energy - 15);
			this.status = 'playing';
			this.playAnimation('playing', () => this.startIdle());
		},

		doPet() {
			if (!this.canAct) return;

			// Check for overpetting
			if (this.petCount >= OVERPET_THRESHOLD) {
				this.happiness = Math.max(0, this.happiness - OVERPET_HAPPINESS_PENALTY);
			} else {
				// Diminishing returns: less effective if overused
				const boost = Math.max(2, 8 - this.petCount * 0.5);
				this.happiness = Math.min(100, this.happiness + boost);
			}
			
			this.petCount++;
			this.status = 'petting';
			this.playAnimation('petting', () => this.startIdle());
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

		restart() {
			this.happiness = 80;
			this.energy = 80;
			this.mood = 80;
			this.petCount = 0;
			this.overfeedCount = 0;
			this.status = 'idle';
			this.startIdle();
			this.startDecay();
		},

		startDecay() {
			this.decayInterval = setInterval(() => {
				if (this.isDead) return;

				if (this.isSleeping) {
					this.energy = Math.min(100, this.energy + SLEEP_REGEN);
				} else {
					// Energy crisis: happiness decays faster when energy is low
					let happinessDecay = HAPPINESS_DECAY;
					if (this.energy < ENERGY_CRISIS_THRESHOLD) {
						// Scale up: at 0 energy, decay is 2x; at 25 energy, decay is 1x
						const factor = 1 + (ENERGY_CRISIS_THRESHOLD - this.energy) / ENERGY_CRISIS_THRESHOLD;
						happinessDecay = HAPPINESS_DECAY * factor;
					}
					this.happiness = Math.max(0, this.happiness - happinessDecay);
					this.energy    = Math.max(0, this.energy - ENERGY_DECAY);
				}

				// Update mood slowly based on happiness
				const moodDiff = this.happiness - this.mood;
				this.mood += moodDiff * MOOD_INERTIA;

				// Handle sickness
				if (this.energy < SICKNESS_THRESHOLD && this.status !== 'sick' && !this.isSleeping) {
					this.startSickness();
				} else if (this.energy >= SICKNESS_THRESHOLD && this.status === 'sick') {
					this.stopSickness();
				}

				// Handle overfeeding recovery (slowly reduce overfeed count)
				if (this.overfeedCount > 0 && this.energy < 90) {
					this.overfeedCount = Math.max(0, this.overfeedCount - 0.1);
				}

				// Handle attention-seeking
				if (this.happiness < ATTENTION_THRESHOLD && this.status !== 'attention' && !this.isSleeping && this.status !== 'sick') {
					this.startAttentionSeeking();
				} else if (this.happiness >= ATTENTION_THRESHOLD && this.status === 'attention') {
					this.stopAttentionSeeking();
				}

				// Reset pet count over time
				if (this.petCount > 0) {
					this.petCount = Math.max(0, this.petCount - 0.2);
				}

				if (this.happiness <= 0 || this.energy <= 0) {
					this.die();
				}
			}, DECAY_INTERVAL);
		},

		// ── Special states ───────────────────────────────────────────────────────
		startSickness() {
			this.stopAnimation();
			this.status = 'sick';
			this.playAnimation('sick');
		},

		stopSickness() {
			this.stopAnimation();
			this.startIdle();
		},

		startAttentionSeeking() {
			this.stopAnimation();
			this.status = 'attention';
			this.playAnimation('attention');

			// Flash screen to get attention
			this.attentionInterval = setInterval(() => {
				if (this.status !== 'attention') {
					clearInterval(this.attentionInterval);
					return;
				}
				// Screen flash effect handled by CSS
			}, 3000);
		},

		stopAttentionSeeking() {
			if (this.attentionInterval) {
				clearInterval(this.attentionInterval);
				this.attentionInterval = null;
			}
			this.stopAnimation();
			this.startIdle();
		},
	},

	created() {
		this.startIdle();
		this.startDecay();
	},

	beforeUnmount() {
		this.stopAnimation();
		clearInterval(this.decayInterval);
		if (this.attentionInterval) clearInterval(this.attentionInterval);
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
}

.frame-screen {
	position: relative;
	display: flex;
	background-color: #c8d8a0;
	border: solid 8px #000000;
	border-radius: 1rem;
	height: 18rem;
	width: 15rem;
	justify-content: center;
	margin-top: 5rem;
	margin-bottom: 3rem;
	box-shadow: inset 0 0 12px rgba(0,0,0,0.25);
	transition: background-color 0.3s ease;
}

.frame-screen.attention-flash {
	background-color: #ffeb3b;
	animation: attention-pulse 0.6s ease;
}

.status-bars-wrapper {
	display: flex;
	flex-direction: row;
	position: absolute;
	width: calc(100% - 1rem);
	margin-top: 0.5rem;
	margin-left: 0.5rem;
	margin-right: 0.5rem;
	gap: 0.5rem;
}

.status-bar {
	display: flex;
	align-items: center;
	justify-content: center;
	height: 1rem;
	border: solid 3px #000000;
	border-radius: 0.3rem;
	font-weight: 700;
	font-size: 0.8rem;
	flex: 1;
	text-align: center;
	position: relative;
	overflow: hidden;
}

.status-bar .bar-value {
	height: 100%;
	position: absolute;
	left: 0;
	top: 0;
	z-index: 2;
	transition: width 0.4s ease, background-color 0.4s ease;
}

.status-bar .bar-label {
	z-index: 3;
	position: relative;
}

.frame-wrapper {
	width: 80%;
	align-self: end;
	margin-bottom: 1rem;
}

.frame-wrapper.is-acting img {
	animation: sprite-bounce 0.35s ease infinite alternate;
}

.frame-wrapper > img {
	width: 100%;
	height: auto;
	image-rendering: crisp-edges;
	filter: grayscale(1);
	object-fit: contain;
}

.button-bar {
	display: flex;
	gap: 0.5rem;
}

.button-bar button {
	all: unset;
	display: inline-flex;
	align-items: center;
	justify-content: center;
	padding: 0.5rem;
	border-radius: 100%;
	background-color: #FFFFFF;
	aspect-ratio: 1/1;
	text-align: center;
	font-family: "Comic Sans MS", "Comic Sans", cursive;
	font-size: 0.9rem;
	font-weight: 900;
	border: solid 5px #000000;
	cursor: pointer;
	user-select: none;
	box-shadow: 1px 1px 5px rgba(0,0,0,0.5);
	transition: background-color 0.15s ease;
}

.button-bar button:hover:not(:disabled) {
	background-color: salmon;
	color: #000000;
	transform: translateY(0.1rem);
	box-shadow: 1px 1px 3px rgba(0,0,0,0.5);
}

.button-bar button:active:not(:disabled) {
	transform: translateY(0.3rem);
	box-shadow: 1px 1px 0 rgba(0,0,0,0.3);
}

.button-bar button:disabled {
	opacity: 0.4;
	cursor: not-allowed;
}

.button-bar button.restart-btn {
	background-color: #e74c3c;
	color: #FFFFFF;
	font-weight: 900;
}

.button-bar button.restart-btn:hover:not(:disabled) {
	background-color: #c0392b;
}

@keyframes sprite-bounce {
	from { transform: translateY(0); }
	to   { transform: translateY(-6px); }
}

@keyframes attention-pulse {
	0%, 100% { background-color: #c8d8a0; }
	50% { background-color: #ffeb3b; }
}

</style>
