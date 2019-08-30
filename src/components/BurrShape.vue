<template>
  <div>
      <Entity v-model="myEntity" v-on:complete="onComplete">
        <Box v-for="vox in voxelPositions" :key="vox.id" ref="myBoxes" :position="vox.position" @complete="onBoxComplete">
        </Box>
      </Entity>  
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
      console.log("onComplete", this._uid)
//      this.reShape() 
      this.buildShape()
    },
    onBoxComplete(evt) {
//      console.log("onBoxComplete", this._uid, this.$refs.myBoxes.length)
    },
    isVisible(dx,dy,dz) { return ( this.stateString[ (dz)*this.x*this.y + (dy)*this.x + (dx) ] == "#") },
    isEmpty(dx,dy,dz) { return ( this.stateString[ (dz)*this.x*this.y + (dy)*this.x + (dx) ] == "_") },
    rotate(idx) { return rotationVector(idx); },
    buildShape(){
      // this implementation does not rely on the BOX meshes being present.
      // It constructs the shape based on the entity string that is passed as prop 
      this.myBoxes=this.$refs.myBoxes

      if (this.theShapeMesh)  { this.theShapeMesh.dispose()}
      var theCSG=BABYLON.CSG.FromMesh(this.myBoxes[0].$entity)

      var bbox=BABYLON.Mesh.CreateBox("bbox",1)
      bbox.parent=this.myEntity
      var dimensions=new BABYLON.Vector3(this.x, this.y, this.z)
      var translation=dimensions.multiplyByFloats(0.5,0.5,0.5)
      translation.addInPlace({x:-0.5, y:-0.5, z:-0.5})
      var delta=-0.05
      dimensions=dimensions.addInPlace({x:delta, y:delta, z:delta})
      bbox.scaling=dimensions
      bbox.position=translation

//      bbox.parent=this.myEntity
//      bbox.computeWorldMatrix()
//      var theCSG = BABYLON.CSG.FromMesh(bbox)

      theCSG.unionInPlace(BABYLON.CSG.FromMesh(bbox))
      bbox.dispose()
      // next punch some holes
      var hole=null
      for (let dx=0; dx < this.x; dx++) {
        for (let dy=0; dy < this.y; dy++) {
          for (let dz=0; dz < this.z; dz++) {
            if (!this.isVisible(dx, dy, dz)) {
                hole=BABYLON.Mesh.CreateBox("hole",1-delta)
                hole.position=new BABYLON.Vector3(dx,dy,dz)
                theCSG.subtractInPlace(BABYLON.CSG.FromMesh(hole))
                hole.dispose()
            }
          }
        }
      }
      // give the ShapeMesh the same parent as the individual boxes
      this.theShapeMesh=theCSG.toMesh({scene: this.myEntity.getScene()})
      this.theShapeMesh.parent=this.myEntity
    },
    reShape() {
//      debugger
      console.log("reShape")
      this.myBoxes=this.$refs.myBoxes
      if (this.theShapeMesh)  { this.theShapeMesh.dispose()}
      // the boxes that make up this shape have now been created.
      // We will merge them into one shape and and hide the individual boxes
      var theCSG=BABYLON.CSG.FromMesh(this.myBoxes[0].$entity)
      this.myBoxes.forEach(box => {
        if (box.$entity)  { 
          theCSG.unionInPlace(BABYLON.CSG.FromMesh(box.$entity)) 
        }
      })
      // make the shape a little bit smaller. Start with the edges.
      var bbox=BABYLON.Mesh.CreateBox("bbox",1)
      bbox.parent=this.myBoxes[0].$entity.parent
      var dimensions=new BABYLON.Vector3(this.x, this.y, this.z)
      var translation=dimensions.multiplyByFloats(0.5,0.5,0.5)
      translation.addInPlace({x:-0.5, y:-0.5, z:-0.5})
      var delta=-0.05
      dimensions=dimensions.addInPlace({x:delta, y:delta, z:delta})
      bbox.scaling=dimensions
      bbox.position=translation
      theCSG.intersectInPlace(BABYLON.CSG.FromMesh(bbox))
      bbox.dispose()
      // next punch some holes
      var hole=null
      for (let dx=0; dx < this.x; dx++) {
        for (let dy=0; dy < this.y; dy++) {
          for (let dz=0; dz < this.z; dz++) {
            if (!this.isVisible(dx, dy, dz)) {
                hole=BABYLON.Mesh.CreateBox("hole",1-delta)
                hole.position=new BABYLON.Vector3(dx,dy,dz)
                theCSG.subtractInPlace(BABYLON.CSG.FromMesh(hole))
                hole.dispose()
            }
          }
        }
      }
      // give the ShapeMesh the same parent as the individual boxes
      this.theShapeMesh=theCSG.toMesh({scene: this.myEntity.getScene()})
      this.theShapeMesh.parent=this.myEntity

      this.myBoxes.forEach(box => {
        if (box.$entity) {
          box.$entity.isVisible=false;
//          box.$entity.scaling=new BABYLON.Vector3(0.98,0.98,0.98)
//          box.$entity.parent=this.theShapeMesh;
          box.$entity.checkCollisions=true;
        }
      })

      // give a material
      /*
      var myMaterial = new BABYLON.StandardMaterial("myMaterial", this.myBoxes[0].$entity.getScene());
      myMaterial.diffuseTexture=new BABYLON.Texture("materials/Wood11_col.jpg", this.myBoxes[0].$entity.getScene())
      myMaterial.diffuseTexture.uScale=0.2
      myMaterial.diffuseTexture.vScale=0.2
      myMaterial.bumpTexture=new BABYLON.Texture("materials/Wood11_nrm.jpg", this.myBoxes[0].$entity.getScene())
      myMaterial.bumpTexture.uScale=0.2
      myMaterial.bumpTexture.vScale=0.2
      myMaterial.specularTexture=new BABYLON.Texture("materials/Wood11_rgh.jpg", this.myBoxes[0].$entity.getScene())
      myMaterial.specularTexture.uScale=0.2
      myMaterial.specularTexture.vScale=0.2
      this.theShapeMesh.material = myMaterial;
      */
    }
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
