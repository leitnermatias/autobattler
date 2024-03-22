<template>
    <div class="board">
        <transition-group name="fade">
            <div class="row" v-for="row, index in board" :key="index">
                <transition-group name="fade">
                    <div  :key="index"
                    class="column" 
                    :class="{targeted: currentTargets.includes(tile), player: tile.isPlayer}"
                    v-for="tile, index in row">
                    <span>
                        {{ tile.emoji }}
                    </span>
                    <transition name="fade">
                        <span class="enemy-health" :key="tile.hp" v-if="currentTargets.includes(tile)">
                            ❤️{{ tile.hp.toFixed(1) }}
                        </span>
                    </transition>
                    </div>
        
                </transition-group>
            </div>
        </transition-group>

    </div>
    <div class="card">
        <p class="title">👾 User stats</p>
        <p>Damage: {{ damage.toFixed(1) }}</p>
        <p>Experience: {{ player.exp }}</p>
        <p>Level: {{ level }}</p>
        <p>Enemies defeated: {{ enemiesDefeated }}</p>
        <p>Targeting {{ Math.ceil(enemiesTargeted) }} enemies</p>
    </div>
    <div class="card upgrades">
        <p class="title">🤩 Upgrades</p>
        <transition-group name="fade">
            <div v-for="upgrade, index in unlockedUpgrades" :key="index" class="upgrade">
                <p class="upgrade-description">
                    {{ upgrade.description }}
                </p>
            </div>
        </transition-group>
    </div>
</template>

<script setup lang="ts">
type BoardTile = {
    hp: number
    emoji: string
    isPlayer?: boolean
    target?: boolean
}

type Upgrade = {
    name: string
    description: string
    unlockedAt: number
    unlocked: boolean
    onActivation: () => void
}

import { computed, onMounted, onUnmounted, ref } from 'vue';
import {emojis} from "../utils"

const mainTick = ref<number>()
const playerTick = ref<number>()
const calculationTick = ref<number>()
const targetingTick = ref<number>()

const player = ref({
    exp: 0
})

const enemiesDefeated = ref(0)

const extraDamage = ref(0)

const enemiesTargeted = ref(1)

const level = computed(() => {
    return Math.floor((player.value.exp / 10))
})

const damage = computed(() => {
    return ((level.value + 1) * 0.10) + extraDamage.value
})

const enemiesCount = computed(() => {
    return level.value >= (props.rows * props.columns) ? (props.rows * props.columns) - 1 : level.value + 1
})

const unlockedUpgrades = computed(() => {
    return upgrades.value.filter(upgrade => upgrade.unlocked)
})

const enemyHealth = computed(() => {
    return (level.value + 1)
})


onMounted(() => {
    calculateBoard()
    mainTick.value = setInterval(() => {
        playerTick.value = setInterval(playerCalculation, 300)
        calculationTick.value = setInterval(gameCalculation, 100)
        targetingTick.value = setInterval(targetingCalculation, 50)
        setTimeout(() => {
            clearInterval(playerTick.value)
            clearInterval(calculationTick.value)
            clearInterval(targetingTick.value)
            currentTargets.value.forEach(tile => tile.target = false)
            currentTargets.value = []
        }, 1900)
        spawnEnemies(enemiesCount.value)
    }, 2000)

})

onUnmounted(() => {
    clearInterval(mainTick.value)
    clearTimeout(playerTick.value)
    clearInterval(calculationTick.value)
    clearInterval(targetingTick.value)
})

const board = ref<BoardTile[][]>([])
const currentTargets = ref<BoardTile[]>([])
const userBoardIndex = ref({
    row: 0,
    col: 0
})

const props = defineProps<{
    columns: number
    rows: number
}>()

function calculateBoard() {
    board.value = []
    for (let i = 0; i < props.rows; i++) {
        const row = []
        for (let j = 0; j < props.columns; j++) {
            row.push({
                emoji: '',
                hp: 10
            })
        }

        board.value.push(row)
    }

    userBoardIndex.value.row = Math.floor(Math.random() * board.value.length)
    const randRow = board.value[userBoardIndex.value.row]
    userBoardIndex.value.col = Math.floor(Math.random() * randRow.length)
    const randCol = randRow[userBoardIndex.value.col]

    randCol.emoji = '👾'
    randCol.isPlayer = true
    
}

function spawnEnemies(amount: number) {

    board.value.forEach((row) => {
        row.forEach((column) => {
            if (!column.isPlayer) {
                column.emoji = ''
                column.hp = 1
            }
        })
    })

    for (let i = 0; i < amount; i++) {
        let rowIndex = Math.floor(Math.random() * board.value.length)
        let randRow = board.value[rowIndex]
        let colIndex = Math.floor(Math.random() * randRow.length)
        let randCol = randRow[colIndex]
        
        while (randCol.isPlayer) {
            rowIndex = Math.floor(Math.random() * board.value.length)
            randRow = board.value[rowIndex]
            colIndex = Math.floor(Math.random() * randRow.length)
            randCol = randRow[colIndex]
        }

        randCol.emoji = emojis[Math.floor(Math.random() * emojis.length)]
        randCol.hp = enemyHealth.value
    }
}

function gameCalculation() {
    upgrades.value
    .filter(upgrade => !upgrade.unlocked)
    .forEach(upgrade => {
        if (upgrade.unlockedAt <= level.value) {
            upgrade.unlocked = true
            upgrade.onActivation()
        }
    })

    currentTargets.value.forEach((tile, index) => {
        if (tile.emoji === '') {
            currentTargets.value.splice(index, 1)
            tile.target = false
        }
    
        if (tile.hp <= 0) {
            tile.emoji = ''
            currentTargets.value.splice(index, 1)
            player.value.exp += 5
            enemiesDefeated.value += 1
            tile.target = false
        } 

    })
}

function playerCalculation() {
    currentTargets.value.forEach(tile => {
        tile.hp -= damage.value
        player.value.exp += 1
    })
}

function targetingCalculation() {
    if (currentTargets.value.length === 0) {
        const tiles = getRandomTiles()
        let count = 0
        tiles.forEach(tile => {
            if (count < enemiesTargeted.value) {
                tile.target = true
                currentTargets.value.push(tile)
                count += 1
            }
        })
    }
}

function getRandomTiles() {
    const enemies: BoardTile[] = []
    board.value.forEach(row => {
        const tiles = row.filter(column => !column.isPlayer && column.emoji !== '' && !column.target)
        if (tiles.length > 0) {
            enemies.push(...tiles)
        }
    })

    
    return enemies
}

const upgrades = ref<Upgrade[]>([
    {
        name: 'My first upgrade',
        description: '+ 0.1 damage',
        unlocked: false,
        unlockedAt: 1,
        onActivation: () => {
            extraDamage.value += 0.1
        }
    },
    {
        name: 'Half a decade',
        description: '+ 0.2 damage',
        unlocked: false,
        unlockedAt: 5,
        onActivation: () => {
            extraDamage.value += 0.2
        }
    },
    {
        name: 'Getting serious',
        description: '+ 0.5 damage',
        unlocked: false,
        unlockedAt: 10,
        onActivation: () => {
            extraDamage.value += 0.5
        }
    },
    {
        name: `What's this? more damage?`,
        description: '+ 0.5 damage',
        unlocked: false,
        unlockedAt: 20,
        onActivation: () => {
            extraDamage.value += 0.7
        }
    },
    {
        name: `100 / 2`,
        description: '+ 1 damage',
        unlocked: false,
        unlockedAt: 50,
        onActivation: () => {
            extraDamage.value += 1
        }
    },
    {
        name: ``,
        description: '+ 2 damage',
        unlocked: false,
        unlockedAt: 75,
        onActivation: () => {
            extraDamage.value += 2
        }
    },
    {
        name: ``,
        description: '+ 2 damage',
        unlocked: false,
        unlockedAt: 100,
        onActivation: () => {
            extraDamage.value += 2
        }
    },
    {
        name: ``,
        description: '+ 5 damage',
        unlocked: false,
        unlockedAt: 125,
        onActivation: () => {
            extraDamage.value += 5
        }
    },
    {
        name: ``,
        description: '+ 5 damage',
        unlocked: false,
        unlockedAt: 150,
        onActivation: () => {
            extraDamage.value += 5
        }
    },
    {
        name: ``,
        description: '+ 7 damage',
        unlocked: false,
        unlockedAt: 175,
        onActivation: () => {
            extraDamage.value += 7
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 250,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 300,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 12 damage',
        unlocked: false,
        unlockedAt: 350,
        onActivation: () => {
            extraDamage.value += 12
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 400,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 15 damage',
        unlocked: false,
        unlockedAt: 500,
        onActivation: () => {
            extraDamage.value += 15
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 600,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 700,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 800,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 900,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 10 damage',
        unlocked: false,
        unlockedAt: 1000,
        onActivation: () => {
            extraDamage.value += 10
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1200,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1300,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1400,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1500,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1600,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1700,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1800,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 1900,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 50,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 100,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
    {
        name: ``,
        description: '+ 1 enemy targeted',
        unlocked: false,
        unlockedAt: 500,
        onActivation: () => {
            enemiesTargeted.value += 1
        }
    },
])
</script>

<style scoped>
</style>