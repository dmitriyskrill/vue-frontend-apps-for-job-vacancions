<template>
  <v-col
      class="align-center px-2 header"
      no-gutters
  >
    <v-col>
      Текущий этаж: {{ currentFloor }}
    </v-col>
    <v-col>
      Скорость: 1 этаж за {{ moveTimePerFloor / 1000 }} секунд
    </v-col>
    <v-col>
      Время отдыха: 1 этаж за {{ relaxTime / 1000 }} секунд(ы)
    </v-col>
    <v-col>
      Очередь вызовов: {{ callStack }}
    </v-col>
  </v-col>
</template>

<script>
export default {
  name: 'ElevatorEmulationSimplifiedEmulatorInfo',
  props: ['currentFloor', 'moveTimePerFloor', 'relaxTime', 'callStack'],
}
</script>

<style scoped>
.header {
  width: 100%;
  height: 100%;
  background: #234908
}
</style>