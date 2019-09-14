<template>
  <Scene v-on:complete="onComplete" v-on:scene="onScene" v-on:engine="onEngine" :environment="{createSkybox: false, createGround: false}">
    <Camera v-model="myCamera" type="arcRotate" :position="[20,15,-10]">
    </Camera>
    <HemisphericLight diffuse="#000" :direction="[0,-1,0]">
      <Property name="groundColor" color="#FFF"/>
    </HemisphericLight>
    <Entity name="myEntity" v-model="myEntity" :activeProblemIndex="0" :rotation="[-Math.PI/2,Math.PI,0]">
      <Box name="myBoundingBox" v-model="myBoundingBox" :scaling="boundingBox.scaling" :position="boundingBox.center">
        <Property name="isVisible" :any="false"/>
      </Box>
      <Entity name="myProblem" v-model="myProblem">
        <BurrProblem ref="someProblem" v-model="initialState" :problem="problems[0]" :entities="shapes">
        </BurrProblem>
      </Entity>
    </Entity>
  </Scene>
</template>

<script>
// The first entity in the scene needs to be the root parent for everything else.
// Its rotation correction is needed to set the starting rotation of the puzzle.
//    <Entity name="myEntity" v-model="myEntity" :activeProblemIndex="0" :rotation="[-Math.PI/2,Math.PI,0]">

var convert = require('xml-js');
var zlib = require('zlib');

//import axios from 'axios'

import Vue from 'vue';
import vb from 'vue-babylonjs';
Vue.use(vb);
import {BABYLON, Scene, Camera, Entity, Box, HemisphericLight, Property} from 'vue-babylonjs';
// import BurrShape from "./BurrShape"
import BurrProblem from "./BurrProblem";
import * as GUI from 'babylonjs-gui';

// import { xml2json } from 'xml-js';

// BABYLON.GUI=GUI

let defaultData = { "puzzle": [{  
                     "shapes": [{ 
                        "voxel" : [{ 
                          "_attributes" : { x: 1, y:1, z:1 , weight: 0, name: "none"},
                          "_text": [ "" ] 
                        }]
                      }], 
                      "problems": [{ 
                        "problem": [{ 
                          "_attributes" : { "name": "none"} ,
                          "shapes": [{ 
                            "shape": [{ 
                              "_attributes": { "count": 0, "id": 0 } 
                            }]  
                          }],
                          "result": [{ 
                            "_attributes": { "count": 0, "id": 0 } 
                          }],
                          "solutions": [{ 
                            "solution": [{
                              "assembly": [{
                                "_text": [ "" ]
                              }],
                              "separation": [{
                                "state": []
                              }]
                            }]
                          }] 
                        }]
                      }] 
                    }]
                  }

export default {
  name: 'HelloWorld',
  components: {BurrProblem, Scene, Camera, Entity, Box, HemisphericLight, Property},
  data() {
    return {
      myFileName: "box.xmpuzzle",
      myEntity: null,
      myCamera: null,
      myProblem: null,
      myBoundingBox: null,
      myPlayMode: true, // allow manual play mode (slide pieces by hand)
      scene: null, // workaround for myScene not being populated by v-model
      jsonFile: defaultData,
      engine: null,
      activeProblemIndex: 0,
      highlightLayer: null,
      myCurrentState: null,
      initialState:"",
      myClickCounter: 0, // used as trigger to make calculated values responsive
      pickedShapes: [],
      selectedShapeId: null,
      myPartitions: []
    }
  },  
  created () {
  // this code is used to load a model from a remote server over http
  // should be replace by some API call and moved to a better place  
/*    axios
    .get('http://localhost:8080/' + this.myFileName, {responseType: 'arraybuffer'})
    .then(response => { 
      var binData = new Uint8Array(response.data);
      var charData = zlib.gunzipSync(Buffer.from(binData))
      this.jsonFile=JSON.parse(convert.xml2json(charData, {nativeType: true, nativeTypeAttributes: true, alwaysArray: true, compact: true}))
    }) */
  },
  computed: {
    complexity() {
      var subComplexity = function(sep) {
        let result=sep.state.length-1
        if (sep.separation != undefined)
          for (let subSep of sep.separation) { result=result + "." + subComplexity(subSep) }
        return result
      }
      return subComplexity(this.jsonFile.puzzle[0].problems[0].problem[0].solutions[0].solution[0].separation[0])
    },
    problems() { 
      return this.jsonFile.puzzle[0].problems[0].problem },
    shapes() { return this.jsonFile.puzzle[0].shapes[0].voxel },
    boundingBox() {
      let sol = this.shapes[this.problems[0].result[0]._attributes.id]
      let mx=Math.max(sol._attributes.x, sol._attributes.y, sol._attributes.z)
      let scaling=[mx, mx, mx]
      let center=[(sol._attributes.x-1)/2, (sol._attributes.y-1)/2, (sol._attributes.z-1)/2]
      return {scaling: scaling, center: center }
    },
    numberOfShapes() { return this.jsonFile.puzzle[0].problems[0].problem[0].solutions[0].solution[0].separation[0].pieces[0]._attributes.count },
    separations() {
      var sepArray=[]
      var separation=this.jsonFile.puzzle[0].problems[0].problem[0].solutions[0].solution[0].separation
      var getStates = function(seps) {
        for (let sep of seps) {
          var allArrays=[]
          var moveArrays=[]
          var jsonArrays=[]
          var move=0
          for (let state of sep.state) {
            var relArray=[]
            var tempArray=new Array
            var piecelist=sep.pieces[0]._text[0]
            let pieces=piecelist.split(" ")
            let dx=state.dx[0]._text[0].split(" ")
            let dy=state.dy[0]._text[0].split(" ")
            let dz=state.dz[0]._text[0].split(" ")
            for (let idx in dx)
            { relArray.push(dx[idx] - dx[0]) // normalize so the first piece is always [0,0,0]
              relArray.push(dy[idx] - dy[0])
              relArray.push(dz[idx] - dz[0])
              let absArray=[]
              absArray.push(dx[idx])
              absArray.push(dy[idx])
              absArray.push(dz[idx])
              tempArray[pieces[idx]]=absArray.join(" ")
            }
            allArrays[move]=relArray.join(" ")
            jsonArrays[move]=tempArray
            moveArrays[allArrays[move]]=move
            move++
          }
          sepArray[piecelist]={separation: sep, states:allArrays, statesPerShape:jsonArrays, moves:moveArrays}
          if (sep.separation != undefined) getStates(sep.separation)
        }
        return sepArray
      }
      return getStates(separation)
    },
    currentStateIndex() {
      var tempArray=[]
        for (let par in this.myPartitions) {
//          console.log("currentStateIndex myCurrentState", this.myCurrentState)
//          console.log("currentStateIndex partition", par)
          if (this.separations[par]) {
//            console.log("currentStateIndex statelookup", this.myCurrentState[par])
//            console.log("currentStateIndex move index", this.separations[par].moves[this.myCurrentState[par]])
            tempArray[par]=this.separations[par].moves[this.myCurrentState[par]]
          }
        }
      return tempArray
    }
  },
  methods: {
    updateCurrentState() {
      // return the StateStrings of the current puzzle taking manual moves into account
      // Array, index is the string of shape ids, value is the string of positions
      this.myCurrentState=[]
      for (let idx in this.myPartitions) // for every partition
      {
        let tempState=[]
        let fsPos=null // first shape position. Needed to normalize (first shape is [0,0,0])
        for (let ss of this.myPartitions[idx]) { // iterate over the Shapes in the partition
          let ssPos = ss.theShapeMesh.getAbsolutePosition() // theShapeMesh absolute position
          let matrix=this.myEntity.getWorldMatrix().invert() 
          let correctedPos=BABYLON.Vector3.TransformCoordinates(ssPos,matrix) // relative position to puzzle container
          correctedPos.x = Math.round(correctedPos.x)
          correctedPos.y = Math.round(correctedPos.y)
          correctedPos.z = Math.round(correctedPos.z)
          if (fsPos == null) fsPos = correctedPos // First Shapes position
          tempState.push(correctedPos.x - fsPos.x) // Shape position relative to first shape
          tempState.push(correctedPos.y - fsPos.y)
          tempState.push(correctedPos.z - fsPos.z)
        }
        this.myCurrentState[idx]=tempState.join(" ")
      }
    },
    calcNextMovingShapes(move) {
      var tmpArray=[]
      for (let par in this.myPartitions) {
        var move=this.currentStateIndex[par]
        if (move >= 0) {
          for (let idx in this.separations[par].statesPerShape[move])
          {
            if (this.separations[par].statesPerShape[move][idx] != this.separations[par].statesPerShape[move+1][idx]) {
              tmpArray.push(idx)
            }
          }
        }
      }
      return tmpArray
    },
    getDragRatio() {
      return this.myCamera.radius/(2*this.scene.getEngine().getRenderingCanvasClientRect().height);
    },
    onEngine(obj) {
      this.engine=obj
    },
    // onScene allows to retrieve the Babylonjs scene object before it is active. 
    // Needed to tweek some settings.
    onScene(obj) { 
      this.scene=obj; //cache the scene in a local object
      this.scene.useRightHandedSystem=true; //set to right handed coordinate system
      this.scene.gravity={x:0, y:0, z:0} // set scene gravity (for collisions)
      // Register callbacks for mouse activity
      this.scene.onPointerDown = this.onPointerDown
      this.scene.onPointerUp = this.onPointerUp
      this.scene.onPointerMove = this.onPointerMove
      this.highlightLayer=new BABYLON.HighlightLayer("highlight", this.scene);
      this.highlightLayer.innerGlow=true
      this.highlightLayer.outerGlow=false
      // test GUI
      var advancedTexture = GUI.AdvancedDynamicTexture.CreateFullscreenUI("ui1", true, this.scene);
      var theStatusPanel = new GUI.StackPanel();
      theStatusPanel.width = "220px";
      theStatusPanel.fontSize = "14px";
      theStatusPanel.color="white"
      theStatusPanel.horizontalAlignment = GUI.Control.HORIZONTAL_ALIGNMENT_LEFT;
      theStatusPanel.verticalAlignment = GUI.Control.VERTICAL_ALIGNMENT_TOP;
      advancedTexture.addControl(theStatusPanel);
      this.guiComplexity=new GUI.TextBlock()
      this.guiComplexity.text="complexity: " + this.complexity
      this.guiComplexity.height="20px"
      theStatusPanel.addControl(this.guiComplexity)
      this.guiCurrentMove=new GUI.TextBlock()
      this.guiCurrentMove.text="current state index: " + this.complexity
      this.guiCurrentMove.height="20px"
      theStatusPanel.addControl(this.guiCurrentMove)
      this.guiHighlightNextMove = new GUI.Checkbox();
      this.guiHighlightNextMove.width = "20px";
      this.guiHighlightNextMove.height = "20px";
      this.guiHighlightNextMove.isChecked = false;
      this.guiHighlightNextMove.color = "green";
      this.guiHighlightNextMove.onIsCheckedChangedObservable.add(this.updateVisuals)
      var panelForCheckbox = GUI.Control.AddHeader(this.guiHighlightNextMove, "Highlight next move", "140px", { isHorizontal: true, controlFirst: true});
      panelForCheckbox.color = "white";
      panelForCheckbox.height = "20px";
      panelForCheckbox.horizontalAlignment = GUI.Control.HORIZONTAL_ALIGNMENT_CENTER;
      theStatusPanel.addControl(panelForCheckbox)
/*      var button1 = GUI.Button.CreateSimpleButton("but1", "Show hint");
      button1.width = 1;
      button1.height = "20px";
      button1.fontSize = 14;
      button1.background = "green";
      //button1.horizontalAlignment = GUI.Control.HORIZONTAL_ALIGNMENT_LEFT;
      theStatusPanel.addControl(button1);
*/    },
    onComplete(obj) {
    },
    onPointerDown(evt, result) {
      switch(true) {
        case ( result.hit && evt.button==0): 
          if (this.myPlayMode == true) {
            if (this.$refs.someProblem) {
              for (let ss of this.$refs.someProblem.$refs.someShapes) { // iterate over the Shapes
                if (ss.theShapeMesh == result.pickedMesh) {
                  this.pickedShapes=[ss.theShapeMesh]
                  this.selectedShapeId=ss.id
                  console.log("onPointerDown on shape id", ss.id)
                }
              }
            }
            this.holdingNormal=result.getNormal(true)
            this.holdingY=this.scene.pointerY
            this.myCamera.detachControl(this.scene.getEngine().getRenderingCanvas())
          }
          break
        case ( !result.hit && evt.button==0):
          this.selectedShapeId=null
          break
        case ( result.hit && evt.button==2):
          if (result.pickedMesh.material.alpha<1)
            result.pickedMesh.material.alpha=1
          else
            result.pickedMesh.material.alpha=0.2
          break
        case ( !result.hit && evt.button==2):
          break
        default:
          break
      }
    },
    onPointerUp(evt, result) {
      // round position to nearest integers in local coordinates
      this.pickedShapes.forEach(function(shape) {
         shape.position=new BABYLON.Vector3(
           Math.round(shape.position.x),
           Math.round(shape.position.y),
           Math.round(shape.position.z)
         )
      })
      this.myCamera.attachControl(this.scene.getEngine().getRenderingCanvas())
      this.pickedShapes = []
      this.calculatePartitions()
      this.updateCurrentState()
    },
    onPointerMove(evt, result) {
      //
      // Shape Dragging logic
      //
      if (this.myPlayMode == true) {
        var maxDelta=0.1 // Maximum distance to travel per cycle. Should be large enough to collide.
        if (this.pickedShapes.length && (this.pickedShapes.length < this.numberOfShapes) ) {
          var delta=Math.sign(this.scene.pointerY-this.holdingY)*Math.min(Math.abs(this.scene.pointerY-this.holdingY)*this.getDragRatio(), maxDelta) // how much did the pointer Y change during drag
          if (delta) {
            this.holdingY = this.scene.pointerY 
            for (let shape of this.pickedShapes) {
              // Move the selected mesh over the corrected diff vector
              shape.translate(this.holdingNormal, delta, BABYLON.Space.WORLD)
              shape.computeWorldMatrix(true)
              // check for collisions
              for (let obstacle of this.$refs.someProblem.$refs.someShapes) {
                let hit=false
                var obstacleMesh=obstacle.theShapeMesh
                if ( !(this.pickedShapes.includes(obstacleMesh))) { // only continue if obstacleMesh is not yet picked
                  // iterate the individual voxels and check for collision
                  for (let pickedVoxel of shape.getChildren()) {
                    pickedVoxel.computeWorldMatrix(true)
                    for (let obstacleVoxel of obstacleMesh.getChildren()) {
                      if (pickedVoxel.intersectsMesh(obstacleVoxel,true)) {
                        hit=true;
                        this.pickedShapes.push(obstacleMesh)
                        break
                      } 
                    }
                    if (hit) break // voxel loop of obstacle
                  }
                }
              }
            }
          }  
        }
      }
    },
    calculatePartitions() {
      var unassignedShapes=[]
      var partitions=[]
      var partitionIds=[]
      var idx=0

      for (let ss of this.$refs.someProblem.$refs.someShapes) {
        unassignedShapes[ss.id]=ss
      }
      this.myPartitions=[]
      for (let ss of unassignedShapes) {
        if (ss != undefined) { 
          partitions[idx]=[ss]
          partitionIds[idx]=[ss.id]
          delete unassignedShapes[ss.id]
          for (let ps of partitions[idx]) {
            for ( let ms of unassignedShapes ) { 
              if ( (ms != undefined) && ps.theShapeMesh.intersectsMesh(ms.theShapeMesh,true)) { 
                partitions[idx].push(ms)
                partitionIds[idx].push(ms.id)
                delete unassignedShapes[ms.id]
              }
            }
          }
          partitionIds[idx]=partitionIds[idx].sort( (a, b) => a - b).join(" ")
          partitions[idx]=partitions[idx].sort( (a, b) => a.id - b.id)
          this.myPartitions[partitionIds[idx]]=partitions[idx]
          idx++
        }
      }
    },
    updateVisuals() {
      var nextMoverIds = this.calcNextMovingShapes(this.currentStateIndex)
      if (this.$refs.someProblem.$refs.someShapes) {
        for (let ss of this.$refs.someProblem.$refs.someShapes) { // iterate over the Shapes
          if (nextMoverIds.includes(ss.id.toString()) && this.guiHighlightNextMove.isChecked) {
              this.highlightLayer.addMesh(ss.theShapeMesh, BABYLON.Color3.Green())
          }
          else {
              this.highlightLayer.removeMesh(ss.theShapeMesh, BABYLON.Color3.Green())
          }
        }
      }
    }
  },
  watch: {
    myCamera() {
      this.scene.activeCamera = this.myCamera
    },
    myBoundingBox() {
      // point the camera at the middle of the boundingbox
      this.myCamera.setTarget(this.myBoundingBox, false, true)
    },
    myFile(newFile, oldFile) {
      var self=this
      if (newFile) {
//        self.jsonFile=defaultData;
        var reader = new FileReader()
        reader.onload = function() {
          var binData = new Uint8Array(reader.result);
          var charData = zlib.gunzipSync(Buffer.from(binData))
          self.jsonFile=JSON.parse(convert.xml2json(charData, {nativeType: true, nativeTypeAttributes: true, alwaysArray: true, compact: true}))
        }
        reader.readAsArrayBuffer(newFile)
      }
    },
    jsonFile(newVal) {
      this.initialState=this.jsonFile.puzzle[0].problems[0].problem[0].solutions[0].solution[0].assembly[0]._text[0]
//      this.updateCurrentState()
      this.guiHighlightNextMove.isChecked = false
      console.log("jsonFile", this.jsonFile, this.initialState)
    },
    complexity() {
      this.guiComplexity.text="complexity: " + this.complexity
    },
    currentStateIndex(newVal,oldVal) {
//      console.log("currentStateIndex watcher")
      this.guiCurrentMove.text="current state index: " + newVal
      this.updateVisuals()
    },
    myCurrentState(newval, oldval) {
//      console.log("myCurrentState", this.myCurrentState)
    },
    separations(newval, oldval) {
//      console.log("separations", newval)
      this.updateCurrentState()
    }
  },
  props: {
    msg: String,
    myFile: null,
  },
}
</script>

<style scoped>

</style>