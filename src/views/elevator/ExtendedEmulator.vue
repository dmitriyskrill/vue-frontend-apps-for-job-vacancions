<template>
  <div class="d-flex align-center justify-center" v-if="elevators.length === 0">Идет загрузка</div>
  <v-row no-gutters v-else class="emulator-container">
    <v-row no-gutters class="main">
      <div class="elevator-emulation-lift-shaft-controller">
        <v-row no-gutters style="height: 100%">
          <elevator-emulation-lift-shaft-controller
              v-for="elevator in elevators"
              :key="elevator.id"
              :elevator="elevator"
              :width="100/elevatorsCount"
              :floorsCount="floorsCount"
              :floors="floors"
              :floorHeight="floorHeight"
          />
        </v-row>
      </div>
      <elevator-emulation-button-block
          class="button-block"
          :floors="floors"
          :floors-count="floorsCount"
          :call-stack="summaryCallStack"
          @setFloor="addFloorToCallStack($event)"
          :floorHeight="floorHeight"
      />
    </v-row>
    <elevator-emulation-extended-emulator-info
        class="extended-emulator-info"
        :elevators="elevators"
        :call-stack="callStack"
        :floorHeight="floorHeight"
    />
  </v-row>
</template>

<script>
import { Elevator } from '@/entities/elevatorEmulator/elevator'
import ElevatorEmulationButtonBlock from '@/components/ElevatorEmulation/ButtonBlock.vue'
import ElevatorEmulationLiftShaft from '@/components/ElevatorEmulation/LiftShaft.vue'
import { mapActions, mapState } from 'pinia'

import { useExtendedElevatorEmulatorStore } from '@/stores/extendedElevatorEmulator'
import ElevatorEmulationLiftShaftController from '@/components/ElevatorEmulation/LiftShaftController.vue'
import ElevatorEmulationExtendedEmulatorInfo from '@/components/ElevatorEmulation/ExtendedEmulatorInfo.vue'

export default {
  name: 'ElevatorExtendedEmulator',
  components: {
    ElevatorEmulationExtendedEmulatorInfo,
    ElevatorEmulationLiftShaftController,
    ElevatorEmulationLiftShaft,
    ElevatorEmulationButtonBlock,
  },
  data: () => ({
    callStackInterval: null,
    cacheInterval: null,
    elevators: [],
    callStack: [],
  }),
  computed: {
    ...mapState(useExtendedElevatorEmulatorStore, ['floorsCount', 'elevatorsCount']),
    floorHeight () {
      const height = this.$vuetify.display.height - 48
      return Math.floor(height / this.floorsCount)
    },
    summaryCallStack () {
      const init = [...this.callStack]
      return this.elevators.reduce((callStack, elevator) => {
        callStack.push(...elevator.callStack)
        return callStack
      }, init)
    },
    floors () {
      const array = []
      for (let i = this.floorsCount; i > 0; i--) {
        array.push(i)
      }
      return array
    },
  },

  methods: {
    ...mapActions(useExtendedElevatorEmulatorStore, ['setFloorsCount', 'setElevatorsCount']),
    addFloorToCallStack (floor) {
      if (this.summaryCallStack.includes(floor)) return
      this.callStack.push(floor)
    },
    addFloorToElevatorCallStack () {
      if (this.callStack.length === 0) return
      const floor = this.callStack[0]

      const goalElevator = this.elevators.find(elevator =>
          (elevator.currentStatus === 'free'
              && elevator.currentFloor === floor)
          || elevator.callStack.includes(floor),
      )
      if (goalElevator) {
        this.callStack.splice(0, 1)
        return
      }

      const freeElevators = this.elevators
          .filter(elevator => elevator.currentStatus === 'free')
          .map(elevator => ({
            elevator,
            difference: Math.abs(floor - elevator.currentFloor),
          }))
          .sort((object1, object2) => {
            const difference1 = object1.difference ? object1.difference : 0
            const difference2 = object2.difference ? object2.difference : 0
            return difference1 > difference2 ? 1 : -1
          })
      //

      // console.log(freeElevators.map(elevator => elevator.difference))
      if (freeElevators[0]) {
        freeElevators[0].elevator.addFloorToCallStack(floor)
        this.callStack.splice(0, 1)
      }

    },
    setToCache () {
      const cacheData = {
        callStack: this.callStack,
        elevators: this.elevators.map(elevator => ({
          id: elevator.id,
          moveTimePerFloor: elevator.moveTimePerFloor,
          relaxTime: elevator.relaxTime,
          moveDirection: elevator.moveDirection,
          callStack: elevator.callStack,
          currentStatus: elevator.currentStatus,
          currentFloor: elevator.currentFloor,
        })),
      }
      localStorage.setItem('multipleElevators', JSON.stringify(cacheData))
      localStorage.setItem('multipleElevatorsSettings', JSON.stringify({
        elevatorsCount: this.elevatorsCount, floorsCount: this.floorsCount,
      }))
    },
    getElevatorsDataFromCache () {
      const cacheData = localStorage.getItem('multipleElevators')
      if (!cacheData || typeof cacheData !== 'string') return null
      return JSON.parse(cacheData)
    },
    setMultipleSettingsFromCache () {
      const multipleElevatorsSettings = localStorage.getItem('multipleElevatorsSettings')
      if (multipleElevatorsSettings && typeof multipleElevatorsSettings === 'string') {
        const settings = JSON.parse(multipleElevatorsSettings)
        this.setFloorsCount(Number.parseInt(settings.floorsCount))
        this.setElevatorsCount(Number.parseInt(settings.elevatorsCount))
      }
    },
    setCallStackInterval () {
      this.callStackInterval = setInterval(this.addFloorToElevatorCallStack, 500)
    },
    clearCallStackInterval () {
      if (!this.callStackInterval) return
      clearInterval(this.callStackInterval)
    },
    setCacheInterval () {
      this.cacheInterval = setInterval(this.setToCache, 1000)
    },
    clearCacheInterval () {
      if (!this.cacheInterval) return
      clearInterval(this.cacheInterval)
    },
    setIntervals () {
      this.setCacheInterval()
      this.setCallStackInterval()
    },
    clearIntervals () {
      this.clearCacheInterval()
      this.clearCallStackInterval()
    },
  },
  created () {
    this.setMultipleSettingsFromCache()
    const elevatorsData = this.getElevatorsDataFromCache()
    if (elevatorsData
        && Array.isArray(elevatorsData.elevators)
        && Array.isArray(elevatorsData.callStack)
    ) {
      const elevators = []
      for (let elevator of elevatorsData.elevators) {
        elevators.push(new Elevator({
          id: elevator.id,
          moveTimePerFloor: elevator.moveTimePerFloor,
          relaxTime: elevator.relaxTime,
          moveDirection: elevator.moveDirection,
          callStack: elevator.callStack,
          currentStatus: elevator.currentStatus,
          currentFloor: elevator.currentFloor,
        }))
      }
      this.elevators = elevators
      this.elevators.forEach(elevator => elevator.initFromCache())
      this.callStack = elevatorsData.callStack
    } else {
      const elevators = []
      for (let i = 1; i <= this.elevatorsCount; i++) {
        elevators.push(new Elevator({ id: i }))
      }
      this.elevators = elevators
    }
  },
  mounted () {
    this.setIntervals()
  },
  beforeRouteLeave () {
    this.clearIntervals()
  },
  beforeRouteUpdate () {
    this.clearIntervals()
  },
}
</script>

<style lang="scss" scoped>
.elevator {
  transition-property: bottom;
  transition-timing-function: linear;
}

.emulator-container {
  height: calc(100vh - 48px);
  width: 100%;

  > .extended-emulator-info {
    width: 40%
  }

  > .main {
    width: 60%;
    height: 100%;
    background: #b3b7ec;

    > .elevator-emulation-lift-shaft-controller {
      width: 90%;
      height: 100%
    }

    > .button-block {
      width: 10%;
      background: #213547;
    }
  }
}

.animated-elevator {
  animation-name: elevator-flashing;
  animation-duration: 500ms;
  animation-iteration-count: infinite;
  animation-direction: alternate;
}

@keyframes elevator-flashing {
  0% {
    opacity: 1;
  }
  100% {
    opacity: 0.5;
  }
}
</style>
