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
        <BurrProblem ref="someProblem" :problem="problems[0]" :entities="shapes">
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
       pickedMeshes: [],
       highlightLayer: null,
       myCurrentState: "",
       myClickCounter: 0 // used as trigger to make calculated values responsive
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
    numberOfShapes() { return this.myProblem.getChildren().length },
    statusArray() {
      // returns an array of State strings. The index is the move number of the solution.
      // a State String is the contatenation of the positions of the puzzle pieces
      var separation=this.jsonFile.puzzle[0].problems[0].problem[0].solutions[0].solution[0].separation[0]
      var allArrays=[]
      var jsonArrays=[]
      var allPieces=[]
      var getStates = function(sep) {
        for (let state of sep.state) {
          let relArray=[]
          let tempArray=new Array
          let pieces=sep.pieces[0]._text[0].split(" ")
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
          allArrays.push(relArray.join(" "))
          jsonArrays.push(tempArray)
          allPieces.push(sep.pieces[0]._text[0])
        }
        if (sep.separation != undefined) getStates(sep.separation[0])
        return {pieces:allPieces, states:allArrays, statesPerShape:jsonArrays}
      }
      return getStates(separation)
    },
    currentStateIndex() {
      return this.statusArray.states.indexOf(this.myCurrentState)
    }
  },
  methods: {
    updateCurrentState() {
      // return the StateString of the current puzzle taking manual moves into account
      console.log("updateCurrentState", this.$refs.someProblem.$refs.someShapes[0].theShapeMesh)
      let tempState=[]
      let fsPos=null // first shape position. Needed to normalize (first shape is [0,0,0])
      if (this.$refs.someProblem) {
        for (let ss of this.$refs.someProblem.$refs.someShapes) { // iterate over the Shapes
          let ssPos = ss.theShapeMesh.getAbsolutePosition() // theShapeMesh
          let matrix=this.myEntity.getWorldMatrix().invert()
          let correctedPos=BABYLON.Vector3.TransformCoordinates(ssPos,matrix)
          correctedPos.x = Math.round(correctedPos.x)
          correctedPos.y = Math.round(correctedPos.y)
          correctedPos.z = Math.round(correctedPos.z)
          if (fsPos == null) fsPos = correctedPos
          tempState.push(correctedPos.x - fsPos.x)
          tempState.push(correctedPos.y - fsPos.y)
          tempState.push(correctedPos.z - fsPos.z)
        }
        this.myCurrentState=tempState.join(" ")
      }
    },
    calcNextMovingShapes(move) {
      var tmpArray=new Array
      for (let idx in this.statusArray.statesPerShape[move]) {
        if (this.statusArray.statesPerShape[move][idx] != this.statusArray.statesPerShape[move+1][idx]) tmpArray.push(idx)
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
            this.pickedMeshes=[result.pickedMesh]
            this.holdingNormal=result.getNormal(true)
            this.holdingY=this.scene.pointerY
            this.myCamera.detachControl(this.scene.getEngine().getRenderingCanvas())
          }
          break
        case ( !result.hit && evt.button==0):
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
      this.pickedMeshes.forEach(function(shape) {
         shape.setPositionWithLocalVector({
           x: Math.round(shape.position.x),
           y: Math.round(shape.position.y),
           z: Math.round(shape.position.z),
         })
      })
      this.myCamera.attachControl(this.scene.getEngine().getRenderingCanvas())
      this.pickedMeshes = []
      this.updateCurrentState()
    },
    onPointerMove(evt, result) {
      //
      // Shape Dragging logic
      //
      if (this.myPlayMode == true) {
        var maxDelta=0.1 // Maximum distance to travel per cycle. Should be large enough to collide.
        if (this.pickedMeshes.length && (this.pickedMeshes.length < this.numberOfShapes) ) {
          var delta=Math.sign(this.scene.pointerY-this.holdingY)*Math.min(Math.abs(this.scene.pointerY-this.holdingY)*this.getDragRatio(), maxDelta) // how much did the pointer Y change during drag
          if (delta) {
            this.holdingY = this.scene.pointerY 
            for (let shape of this.pickedMeshes) {
              // Move the selected mesh over the corrected diff vector
              shape.translate(this.holdingNormal, delta, BABYLON.Space.WORLD)
              shape.computeWorldMatrix(true)
              // check for collisions
              for (let obstacle of this.myProblem.getChildren()) {
                let hit=false
                var obstacleMesh=obstacle.getChildren()[0]
                if ( !(this.pickedMeshes.includes(obstacleMesh))) { // only continue if obstacleMesh is not yet picked
                  // iterate the individual voxels and check for collision
                  for (let pickedVoxel of shape.getChildren()) {
                    pickedVoxel.computeWorldMatrix(true)
                    for (let obstacleVoxel of obstacleMesh.getChildren()) {
                      if (pickedVoxel.intersectsMesh(obstacleVoxel,true)) {
                        hit=true;
                        this.pickedMeshes.push(obstacleMesh)
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
        else {
        }
      }
    },
    updateVisuals() {
//      console.log("updateVisuals", this.myCurrentState)
      if (this.currentStateIndex >= 0) {
        for (let idx of this.calcNextMovingShapes(this.currentStateIndex)) {
          if (this.$refs.someProblem.myShapeEntities[idx] != undefined) {
            if (this.guiHighlightNextMove.isChecked) {
              this.highlightLayer.addMesh(this.$refs.someProblem.myShapeEntities[idx].getChildMeshes(true)[0], BABYLON.Color3.Green())
            }
            else
              this.highlightLayer.removeMesh(this.$refs.someProblem.myShapeEntities[idx].getChildMeshes(true)[0], BABYLON.Color3.Green())
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
        self.jsonFile=defaultData;
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
      this.myCurrentState=this.statusArray.states[0]
      this.guiHighlightNextMove.isChecked = false
    },
    complexity() {
      this.guiComplexity.text="complexity: " + this.complexity
    },
    currentStateIndex(newVal,oldVal) {
      this.guiCurrentMove.text="current state index: " + newVal
      for (let idx of this.calcNextMovingShapes(oldVal)) {
        if (this.$refs.someProblem.myShapeEntities[idx] != undefined)
          this.highlightLayer.removeMesh(this.$refs.someProblem.myShapeEntities[idx].getChildMeshes(true)[0], BABYLON.Color3.Green())
      }
      if ( (newVal >=0) && (this.statusArray.pieces[newVal].length > this.statusArray.pieces[newVal+2].length) )
        console.log("removable piece")
      this.updateVisuals()
    },
    myCurrentState(newVal, oldVal) {
    },
  },
  props: {
    msg: String,
    myFile: null,
  },
}
</script>

<style scoped>

</style>