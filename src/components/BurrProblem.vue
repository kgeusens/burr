<template>
  <div>
    <div v-for="shape in shapes" :key="shape.id">
      <Entity>
        <BurrShape ref="someShapes" :entity="shape.entity" :id="shape.id" :shapePosition="shape.position" :shapeRotation="rotate(shape.rotationIndex)">
        </BurrShape>
      </Entity>
    </div>
  </div>
</template>

<script>

import {Entity} from 'vue-babylonjs';
import BurrShape from "./BurrShape.vue"
import { rotationVector } from "./burrUtils.js";


export default {
  name: 'BurrProblem',
  components: {BurrShape, Entity},
  props: 
  {
    problem: Object,
    entities: Array,
    value: String // This is the String of positions (status) using v-model
  }, 
  data() {
    return {
      id: null,
      currentSolution: 0,
    }
  },
  mounted () {
    this.id = this._uid
  },
  methods: {
    rotate(idx) { return rotationVector(idx); },
    getCurrentState() {
      // return the StateString of the current puzzle taking manual moves into account
      let tempState=[]
      let fsPos=null // first shape position. Needed to normalize (first shape is [0,0,0])
      for (let ss of this.$refs.someShapes) { // iterate over the Shapes
        let ssPos = ss.theShapeMesh.position // theShapeMesh
        if (fsPos == null) fsPos = ssPos
        tempState.push(ssPos.x - fsPos.x)
        tempState.push(ssPos.y - fsPos.y)
        tempState.push(ssPos.z - fsPos.z)
      }
      return tempState.join(" ")
    }
  },
  computed: {
    shapes() {
      // returns an array that maps to the id of the unique entities. 
      // A shape with id=2 and count=3 means entity 2 is used three times. 
      // It will occur as ... 2, 2, 2 ... in the array
      // We return an array of entities, together with its position, and the rotation
      var shapeList = []
      var tempShapes = []
      var assembly = this.problem.solutions[0].solution[this.currentSolution].assembly[0]._text[0].split(" ")
      for (let shape of this.problem.shapes[0].shape) {
        tempShapes[shape._attributes.id] = shape._attributes.count
      }
      let j = 0
      for (let idx=0; idx < tempShapes.length; idx++) {
        for (let i=0; i<tempShapes[idx]; i++) {
          shapeList.push( { entity: this.entities[idx] , id: j, rotationIndex: assembly[j*4+3], position: [Number(assembly[j*4]), Number(assembly[j*4+1]), Number(assembly[j*4+2]) ]} )
          j++
        }
      }
      return shapeList
    },
    positions() {
      var posList = []
      var vals=this.value.split(" ")
      let i=0
      while ( i<vals.length) {
        posList.push( [Number(vals[i]), Number(vals[i+1]), Number(vals[i+2])] )
        i+=3
      }
      console.log(posList)
      return posList
    }
  },
  watch: {
    value() {
      console.log("state changed", this.value)
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

</style>
