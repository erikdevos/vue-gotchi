# Vue Gotchi

A digital Tamagotchi-style virtual pet built with Vue.js. Care for your gotchi by managing its happiness and energy levels through various interactions.

## Game Mechanics

### Core Stats
- **Happiness** (0-100): Decreases over time, affected by interactions
- **Energy** (0-100): Decreases over time, affects happiness decay rate
- **Mood** (0-100): Changes slowly based on happiness, affects idle animations

### Actions & Effects
- **Eat**: +25 energy, +5 happiness
- **Play**: +20 happiness, -15 energy  
- **Pet**: +2-8 happiness (diminishing returns), no energy cost
- **Sleep**: Energy regens +8/tick, happiness doesn't decay

### Special Conditions

#### Energy Crisis
- Below 25 energy: Happiness decays 1-2x faster
- Below 15 energy: Gotchi becomes sick (mad→sad→mad animation)

#### Attention Seeking
- Below 35 happiness: Gotchi seeks attention (mad→sad→mad→neutral animation)
- Screen flashes yellow every 3 seconds
- Blocks other actions until happiness improves

#### Mood System
- Mood changes slowly based on happiness (inertia factor: 0.1)
- Different idle animations based on mood:
  - 80+ mood: Happy idle (happy→joy→happy→calm)
  - 50-79 mood: Normal idle (calm→casual→neutral→calm)
  - 25-49 mood: Sad idle (sad→neutral→sad→calm)
  - <25 mood: Mad idle (mad→sad→mad→neutral)

#### Overfeeding
- Feeding at 100% energy: Escalating happiness penalty (5, 7, 9, 11, 13...)
- 4+ overfeeds: Triggers sickness immediately
- Overfeed count recovers slowly when energy < 90%
- No instant death - but can lead to death via happiness depletion

#### Overpetting
- 8+ pets in succession: -3 happiness per pet
- Below threshold: Diminishing returns (8→2 happiness boost)
- Pet count recovers by 0.2 per tick automatically

#### Death Conditions
- Happiness reaches 0
- Energy reaches 0
- Death animation: shocked→sad loop
- Red "Restart" button appears to reset the game

### Animation System
All animations are configured at the top of `TamagotchiDevice.vue` for easy editing:
- Frame sequences map to `/assets/sprites/sprite_<name>.png`
- Adjustable timing per animation
- Loop vs one-shot animations

### Game Loop
- Stats update every 6 seconds (configurable)
- Energy crisis affects happiness decay rate
- Mood follows happiness with inertia
- Various conditions trigger automatically based on stat thresholds

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).
