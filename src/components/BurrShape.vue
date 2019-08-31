<template>
  <div>
      <Box v-model="theShapeMesh" v-on:complete="onComplete">
        <Box v-for="vox in voxelPositions" :key="vox.id" ref="myBoxes" :position="vox.position" @complete="onBoxComplete">
        </Box>
      </Box>  
  </div>
</template>

<script>

import { rotationVector } from "./burrUtils.js";
import { BABYLON, Entity, Box } from 'vue-babylonjs';

export default {
  name: 'BurrShape',
  components: { Box, Entity },
  props: 
  {
    entity: null
  }, 
  data() {
    return {
      id: null,
      myEntity: null,
      theShapeMesh: null
    }
  },
  mounted () {
    this.id = this._uid
  },
  beforeDestroy() {
    if (this.theShapeMesh) this.theShapeMesh.dispose()
  },
  updated() {
//    console.log("updated", this._uid)
//    this.buildShape()
  },
  methods: {
    onComplete() {
      console.log("onComplete", this._uid, this.theShapeMesh)
      this.buildShapeMesh()
    },
    onBoxComplete(evt) {
      console.log("onBoxComplete", this._uid, this.$refs.myBoxes.length)
      evt.entity.isVisible=false;
      evt.entity.scaling=new BABYLON.Vector3(0.98,0.98,0.98)
//      evt.entity.parent=this.theShapeMesh;
      evt.entity.checkCollisions=true;
    },
    isVisible(dx,dy,dz) { return ( this.stateString[ (dz)*this.x*this.y + (dy)*this.x + (dx) ] == "#") },
    isEmpty(dx,dy,dz) { return ( this.stateString[ (dz)*this.x*this.y + (dy)*this.x + (dx) ] == "_") },
    rotate(idx) { return rotationVector(idx); },
    buildShapeMesh(){
      // this implementation does not rely on the BOX meshes being present.
      // It constructs the shape based on the entity string that is passed as prop 

      // Build the shape using CSG only
      // Calculate the vertices of the bounding box starting from a standard box (default of theShapeMesh)
      var vertexData = BABYLON.VertexData.ExtractFromMesh(this.theShapeMesh);
      var dimensions=new BABYLON.Vector3(this.x, this.y, this.z)
      var translation=dimensions.multiplyByFloats(0.5,0.5,0.5)
      var delta=-0.05
      for (let idx=0; idx < (vertexData.positions.length / 3); idx++)
      {
        vertexData.positions[idx*3] *= dimensions.x + delta
        vertexData.positions[1+idx*3] *= dimensions.y + delta
        vertexData.positions[2+idx*3] *= dimensions.z + delta
        vertexData.positions[idx*3] = Math.round((vertexData.positions[idx*3] + translation.x - 0.5)*1000)/1000
        vertexData.positions[1+idx*3] = Math.round((vertexData.positions[1+idx*3] + translation.y - 0.5)*1000)/1000
        vertexData.positions[2+idx*3] = Math.round((vertexData.positions[2+idx*3] + translation.z - 0.5)*1000)/1000
      }
      vertexData.applyToMesh(this.theShapeMesh, true)
      // next use some CSG magic to punch holes in the bounding box, leaving the shape
      var theCSG=BABYLON.CSG.FromMesh(this.theShapeMesh)       
      var hole=BABYLON.Mesh.CreateBox("hole",1-delta)
      for (let dx=0; dx < this.x; dx++) {
        for (let dy=0; dy < this.y; dy++) {
          for (let dz=0; dz < this.z; dz++) {
            if (!this.isVisible(dx, dy, dz)) {
                hole.position=new BABYLON.Vector3(dx,dy,dz)
                theCSG.subtractInPlace(BABYLON.CSG.FromMesh(hole))
            }
          }
        }
      }
      hole.dispose()
      // update the vertices in theShapeMesh based on the final CSG
      var tempMesh=theCSG.toMesh({scene: this.theShapeMesh.getScene()})
      vertexData = BABYLON.VertexData.ExtractFromMesh(tempMesh);
      vertexData.applyToMesh(this.theShapeMesh)
      tempMesh.dispose()
      // Give the mesh some Material
      var myMaterial = new BABYLON.StandardMaterial("myMaterial", this.theShapeMesh.getScene());
      this.theShapeMesh.material = myMaterial;
    },
  },
  computed: {
    stateString() {
      return this.entity._text[0].replace(/\d+/g,""); //remove color code
    },
    voxelPositions() {
      var voxArr = new Array();
      var theId = 0
      for (let dx=0; dx < this.x; dx++) {
        for (let dy=0; dy < this.y; dy++) {
          for (let dz=0; dz < this.z; dz++) {
            if (this.isVisible(dx, dy, dz)) { voxArr.push({ position: [dx, dy, dz], id:theId }); theId++ }
          }
        }
      }
      return voxArr
    },
    x() { return this.entity._attributes.x},
    y() { return this.entity._attributes.y},
    z() { return this.entity._attributes.z},
    name() { return this.entity._attributes.name},
  },
  watch: {
    voxelPositions(newval,oldval) {
      console.log("watch voxelPositions")
    },
    myEntity() {
      console.log("watch myEntity")
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

</style>
