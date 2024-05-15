<script>
  import { afterUpdate, onDestroy } from "svelte";
  import * as THREE from "three";
  import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
  import TextSprite from "@seregpie/three.text-sprite";
  import * as math from "mathjs";
  import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
  import backgroundNebulaNX from "../assets/textures/backgroundNebulanx.png";
  import backgroundNebulaPX from "../assets/textures/backgroundNebulapx.png";
  import backgroundNebulaNY from "../assets/textures/backgroundNebulany.png";
  import backgroundNebulaPY from "../assets/textures/backgroundNebulapy.png";
  import backgroundNebulaNZ from "../assets/textures/backgroundNebulanz.png";
  import backgroundNebulaPZ from "../assets/textures/backgroundNebulapz.png";

  const spaceUrl = new URL("../assets/models/space.glb", import.meta.url);
  const assetLoader = new GLTFLoader();
  const cubetextureloader = new THREE.CubeTextureLoader();

  function loadModel(url, name, scene=undefined) {
      return new Promise((resolve, reject) => {
          assetLoader.load(url.href, function(gltf) {
              const model = gltf.scene;
              if (scene != undefined) {
                scene.add(model);
              }
              model.name = name;

              model.traverse((child) => {
                  if (child.isMesh) {
                      child.castShadow = true;
                      child.receiveShadow = true;
                  }
              });

              resolve(model);
          }, undefined, function(error) {
              reject(error);
          });
      });
  }

  let inputObj = {state: {'0': '1', '1': '0'}, transformMatrix: {'0-0': '1', '0-1': '0', '1-0': '0', '1-1': '1'}, toBlochSphereAnimation: false, addMeasurementAnimation: false, addQubitParticleAnimation: false, measurementBasis: 'Z', qubitParticle: false};

  let scene;

  let arrowVertAngleArr = [];
  let dotsVertAngleArr = [];

  let curStateObj = {main: undefined, arrowObjArr: [], dotObjArr: []};

  function reset() {
    inputObj = {state: {'0': '1', '1': '0'}, transformMatrix: {'0-0': '1', '0-1': '0', '1-0': '0', '1-1': '1'}, toBlochSphereAnimation: false, addMeasurementAnimation: false, addQubitParticleAnimation: false, measurementBasis: 'Z', qubitParticle: false};

    scene.clear();

    arrowVertAngleArr = [];
    dotsVertAngleArr = [];

    curStateObj = {main: undefined, arrowObjArr: [], dotObjArr: []};

    initScene();
  }

  function toBlochSphereClicked() {
    inputObj.toBlochSphereAnimation = true;
  }

  function addStateClicked(basis) {
    if (curStateObj.main.circleXY == undefined) {
      const arrow = addStateToRealPlane(inputObj.state['0'], inputObj.state['1'], basis, 0xff0000);
      curStateObj.arrowObjArr.push(arrow);
    }
    else {
      const arrow = addStateToBlochSphere(inputObj.state['0'], inputObj.state['1'], basis, 0xff0000);
      curStateObj.arrowObjArr.push(arrow);
    }
    curStateObj = curStateObj;
  }

  function addQubitParticleClicked(state) {
    inputObj.addQubitParticleAnimation = true;
    inputObj.qubitParticle = state;
  }

  function applyGateClicked() {
    const arrowObj = curStateObj.arrowObjArr[0];
    const label = arrowObj.labelText; //"⎧"+toOutput[0]+"⎫\n⎩"+toOutput[1]+"⎭"
    let alphaAndBeta = label.replace(/[⎧⎫⎩⎭]/g, '').split("\n");
    curStateObj.arrowObjArr = [];
    removeGroups([arrowObj]);
    alphaAndBeta = applyTransform(alphaAndBeta[0], alphaAndBeta[1],
      [[inputObj.transformMatrix['0-0'],inputObj.transformMatrix['0-1']],
      [inputObj.transformMatrix['1-0'],inputObj.transformMatrix['1-1']]]);

    if (curStateObj.main.circleXY == undefined) {
      curStateObj.arrowObjArr.push(addStateToRealPlane(alphaAndBeta['0'], alphaAndBeta['1'], 'Z', 0xff0000));
    }
    else {
      curStateObj.arrowObjArr.push(addStateToBlochSphere(alphaAndBeta['0'], alphaAndBeta['1'], 'Z', 0xff0000));
    }
    curStateObj = curStateObj;
  }

  function addMeasurementDotsClicked(basis) {
    if (curStateObj.main.circleXY == undefined) {
      const arrowObj = curStateObj.arrowObjArr[0];
      const dotsArr = addMeasurementDots(arrowObj.vertAngle, 2*arrowObj.radius, arrowObj.color, basis);
      curStateObj.dotObjArr = curStateObj.dotObjArr.concat(dotsArr);
    }
    curStateObj = curStateObj;
  }

  function addPhaseDotsClicked() {
    if (curStateObj.main.circleXY == undefined) {
      const arrowObj = curStateObj.arrowObjArr[0];
      const dotsArr = addGlobalPhaseDots(arrowObj.vertAngle, 2*arrowObj.radius, arrowObj.color);
      curStateObj.dotObjArr = curStateObj.dotObjArr.concat(dotsArr);
    }
    curStateObj = curStateObj;
  }

  function addMeasurementClicked(basis='Z') {
    inputObj.measurementBasis = basis;
    inputObj.addMeasurementAnimation = true;
  }

  function addBlochSphere() {
    const blochSphereObj = {};

    const circleGeometryXY = new THREE.TorusGeometry(10, 0.1, 3, 100, 2*Math.PI);
    const circleMaterialXY = new THREE.MeshStandardMaterial({color: 0x000000});
    const circleXY = new THREE.Mesh(circleGeometryXY, circleMaterialXY);
    scene.add(circleXY);
    blochSphereObj.circleXY = circleXY;

    const circleGeometryXZ = new THREE.TorusGeometry(10, 0.1, 3, 100, 2*Math.PI);
    const circleMaterialXZ = new THREE.MeshStandardMaterial({color: 0x000000});
    const circleXZ = new THREE.Mesh(circleGeometryXZ, circleMaterialXZ);
    circleXZ.rotateY(Math.PI/2);
    scene.add(circleXZ);
    blochSphereObj.circleXZ = circleXZ;

    const circleGeometryYZ = new THREE.TorusGeometry(10, 0.1, 3, 100, 2*Math.PI);
    const circleMaterialYZ = new THREE.MeshStandardMaterial({color: 0x000000});
    const circleYZ = new THREE.Mesh(circleGeometryYZ, circleMaterialYZ);
    circleYZ.rotateX(Math.PI/2);
    scene.add(circleYZ);
    blochSphereObj.circleYZ = circleYZ;

    const lineGeometryX = new THREE.CylinderGeometry(0.1, 0.1, 24, 3, 1);
    const lineMaterialX = new THREE.MeshStandardMaterial({color: 0x000000});
    const lineX = new THREE.Mesh(lineGeometryX, lineMaterialX);
    lineX.rotateZ(Math.PI/2);
    scene.add(lineX);
    blochSphereObj.lineX = lineX;

    const arrowGeometryX0 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialX0 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowX0 = new THREE.Mesh(arrowGeometryX0, arrowMaterialX0);
    arrowX0.translateX(12);
    arrowX0.rotateZ(-Math.PI/2);
    scene.add(arrowX0);
    blochSphereObj.arrowX0 = arrowX0;

    const arrowGeometryX1 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialX1 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowX1 = new THREE.Mesh(arrowGeometryX1, arrowMaterialX1);
    arrowX1.translateX(-12);
    arrowX1.rotateZ(Math.PI/2);
    scene.add(arrowX1);
    blochSphereObj.arrowX1 = arrowX1;

    const lineGeometryY = new THREE.CylinderGeometry(0.1, 0.1, 24, 3, 1);
    const lineMaterialY = new THREE.MeshStandardMaterial({color: 0x000000});
    const lineY = new THREE.Mesh(lineGeometryY, lineMaterialY);
    scene.add(lineY);
    blochSphereObj.lineY = lineY;

    const arrowGeometryY0 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialY0 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowY0 = new THREE.Mesh(arrowGeometryY0, arrowMaterialY0);
    arrowY0.translateY(12);
    scene.add(arrowY0);
    blochSphereObj.arrowY0 = arrowY0;

    const arrowGeometryY1 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialY1 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowY1 = new THREE.Mesh(arrowGeometryY1, arrowMaterialY1);
    arrowY1.translateY(-12);
    arrowY1.rotateX(Math.PI);
    scene.add(arrowY1);
    blochSphereObj.arrowY1 = arrowY1;

    const lineGeometryZ = new THREE.CylinderGeometry(0.1, 0.1, 24, 3, 1);
    const lineMaterialZ = new THREE.MeshStandardMaterial({color: 0x000000});
    const lineZ = new THREE.Mesh(lineGeometryZ, lineMaterialZ);
    lineZ.rotateX(Math.PI/2);
    scene.add(lineZ);
    blochSphereObj.lineZ = lineZ;

    const arrowGeometryZ0 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialZ0 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowZ0 = new THREE.Mesh(arrowGeometryZ0, arrowMaterialZ0);
    arrowZ0.translateZ(12);
    arrowZ0.rotateX(Math.PI/2);
    scene.add(arrowZ0);
    blochSphereObj.arrowZ0 = arrowZ0;

    const arrowGeometryZ1 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialZ1 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowZ1 = new THREE.Mesh(arrowGeometryZ1, arrowMaterialZ1);
    arrowZ1.translateZ(-12);
    arrowZ1.rotateX(-Math.PI/2);
    scene.add(arrowZ1);
    blochSphereObj.arrowZ1 = arrowZ1;

    let labelX0 = new TextSprite({
        text: '|𝑖⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelX0.translateX(14);
    scene.add(labelX0);
    blochSphereObj.labelX0 = labelX0;
    
    let labelX1 = new TextSprite({
        text: '|-𝑖⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelX1.translateX(-14);
    scene.add(labelX1);
    blochSphereObj.labelX1 = labelX1;

    let labelY0 = new TextSprite({
        text: '|0⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelY0.translateY(14);
    scene.add(labelY0);
    blochSphereObj.labelY0 = labelY0;

    let labelY1 = new TextSprite({
        text: '|1⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelY1.translateY(-14);
    scene.add(labelY1);
    blochSphereObj.labelY1 = labelY1;

    let labelZ0 = new TextSprite({
        text: '|+⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelZ0.translateZ(14);
    scene.add(labelZ0);
    blochSphereObj.labelZ0 = labelZ0;
    
    let labelZ1 = new TextSprite({
        text: '|-⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelZ1.translateZ(-14);
    scene.add(labelZ1);
    blochSphereObj.labelZ1 = labelZ1;

    return blochSphereObj;
  }

  function addRealPLane() {
    const realPlaneObj = {};

    const circleGeometryYZ = new THREE.TorusGeometry(10, 0.1, 3, 100, 2*Math.PI);
    const circleMaterialYZ = new THREE.MeshStandardMaterial({color: 0x000000});
    const circleYZ = new THREE.Mesh(circleGeometryYZ, circleMaterialYZ);
    circleYZ.rotateY(Math.PI/2);
    scene.add(circleYZ);
    realPlaneObj.circleYZ = circleYZ;

    const lineGeometryY0 = new THREE.CylinderGeometry(0.1, 0.1, 24, 3, 1);
    const lineMaterialY0 = new THREE.MeshStandardMaterial({color: 0x000000});
    const lineY0 = new THREE.Mesh(lineGeometryY0, lineMaterialY0);
    lineY0.rotateX(Math.PI/2);
    scene.add(lineY0);
    realPlaneObj.lineY0 = lineY0;

    const arrowGeometryY0 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialY0 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowY0 = new THREE.Mesh(arrowGeometryY0, arrowMaterialY0);
    arrowY0.translateZ(12);
    arrowY0.rotateX(Math.PI/2);
    scene.add(arrowY0);
    realPlaneObj.arrowY0 = arrowY0;

    const lineGeometryY1 = new THREE.CylinderGeometry(0.1, 0.1, 24, 3, 1);
    const lineMaterialY1 = new THREE.MeshStandardMaterial({color: 0x000000});
    const lineY1 = new THREE.Mesh(lineGeometryY1, lineMaterialY1);
    scene.add(lineY1);
    realPlaneObj.lineY1 = lineY1;

    const arrowGeometryY1 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialY1 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowY1 = new THREE.Mesh(arrowGeometryY1, arrowMaterialY1);
    arrowY1.translateY(12);
    scene.add(arrowY1);
    realPlaneObj.arrowY1 = arrowY1;

    const lineGeometryZ0 = new THREE.CylinderGeometry(0.1, 0.1, 12, 3, 1);
    const lineMaterialZ0 = new THREE.MeshStandardMaterial({color: 0x000000});
    const lineZ0 = new THREE.Mesh(lineGeometryZ0, lineMaterialZ0);
    lineZ0.translateY(6*Math.cos(Math.PI/4));
    lineZ0.translateZ(6*Math.sin(Math.PI/4))
    lineZ0.rotateX(Math.PI/4);
    scene.add(lineZ0);
    realPlaneObj.lineZ0 = lineZ0;

    const arrowGeometryZ0 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialZ0 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowZ0 = new THREE.Mesh(arrowGeometryZ0, arrowMaterialZ0);
    arrowZ0.translateY(12*Math.cos(Math.PI/4));
    arrowZ0.translateZ(12*Math.sin(Math.PI/4))
    arrowZ0.rotateX(Math.PI/4);
    scene.add(arrowZ0);
    realPlaneObj.arrowZ0 = arrowZ0;

    const lineGeometryZ1 = new THREE.CylinderGeometry(0.1, 0.1, 12, 3, 1);
    const lineMaterialZ1 = new THREE.MeshStandardMaterial({color: 0x000000});
    const lineZ1 = new THREE.Mesh(lineGeometryZ1, lineMaterialZ1);
    lineZ1.translateY(6*Math.cos(3*Math.PI/4));
    lineZ1.translateZ(6*Math.sin(3*Math.PI/4))
    lineZ1.rotateX(3*Math.PI/4);
    scene.add(lineZ1);
    realPlaneObj.lineZ1 = lineZ1;

    const arrowGeometryZ1 = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterialZ1 = new THREE.MeshStandardMaterial({color: 0x000000});
    const arrowZ1 = new THREE.Mesh(arrowGeometryZ1, arrowMaterialZ1);
    arrowZ1.translateY(12*Math.cos(3*Math.PI/4));
    arrowZ1.translateZ(12*Math.sin(3*Math.PI/4))
    arrowZ1.rotateX(3*Math.PI/4);
    scene.add(arrowZ1);
    realPlaneObj.arrowZ1 = arrowZ1;

    let labelY0 = new TextSprite({
        text: '|0⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelY0.translateZ(14);
    scene.add(labelY0);
    realPlaneObj.labelY0 = labelY0;

    let labelY1 = new TextSprite({
        text: '|1⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelY1.translateY(14);
    scene.add(labelY1);
    realPlaneObj.labelY1 = labelY1;

    let labelZ0 = new TextSprite({
        text: '|+⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelZ0.translateY(14*Math.cos(Math.PI/4));
    labelZ0.translateZ(14*Math.sin(Math.PI/4))
    scene.add(labelZ0);
    realPlaneObj.labelZ0 = labelZ0;
    
    let labelZ1 = new TextSprite({
        text: '|-⟩',
        fontSize: 2,
        color: '#000000',
    });
    labelZ1.translateY(14*Math.cos(3*Math.PI/4));
    labelZ1.translateZ(14*Math.sin(3*Math.PI/4))
    scene.add(labelZ1);
    realPlaneObj.labelZ1 = labelZ1;

    return realPlaneObj;
  }

  function addArrow(horizAngle=-Math.PI/2, vertAngle, radius=10, color=0xff0000, fromCenter=0, labelText, sideLabel = false) {
    const arrowObj = {horizAngle, vertAngle, radius, color, fromCenter, labelText};

    const lineGeometry = new THREE.CylinderGeometry(0.2, 0.2, radius*2-1, 3, 1);
    const lineMaterial = new THREE.MeshStandardMaterial({color: color});
    const line = new THREE.Mesh(lineGeometry, lineMaterial);
    line.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
    line.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
    line.translateY(-0.5 + fromCenter);
    scene.add(line);
    arrowObj.line = line;

    const arrowGeometry = new THREE.ConeGeometry(0.4, 1, 10, 1);
    const arrowMaterial = new THREE.MeshStandardMaterial({color: color});
    const arrow = new THREE.Mesh(arrowGeometry, arrowMaterial);
    arrow.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
    arrow.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
    arrow.translateY(radius-0.5 + fromCenter);
    scene.add(arrow);
    arrowObj.arrow = arrow;

    let colorString = color.toString(16);
    while (colorString.length < 6) {
      colorString = "0" + colorString;
    }
    colorString = "#" + colorString;

    let label = new TextSprite({
        text: labelText,
        fontSize: 1,
        color: colorString,
        fontFamily: "monospace",
    });
    label.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
    label.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
    if (sideLabel) {
      label.translateY(fromCenter);
      label.translateX(2);
    }
    else {
      label.translateY(1.4*(radius + fromCenter));
    }
    scene.add(label);
    arrowObj.label = label;

    return arrowObj;
  }

  function changeAxisColor(axisString, color, blochSphereObj) {
    if (axisString == "x") {
      blochSphereObj.lineX.material.color = new THREE.Color(color);
      blochSphereObj.arrowX0.material.color = new THREE.Color(color);
      blochSphereObj.arrowX1.material.color = new THREE.Color(color);
    }
    if (axisString == "y") {
      blochSphereObj.lineY.material.color = new THREE.Color(color);
      blochSphereObj.arrowY0.material.color = new THREE.Color(color);
      blochSphereObj.arrowY1.material.color = new THREE.Color(color);
    }
    if (axisString == "z") {
      blochSphereObj.lineZ.material.color = new THREE.Color(color);
      blochSphereObj.arrowZ0.material.color = new THREE.Color(color);
      blochSphereObj.arrowZ1.material.color = new THREE.Color(color);
    }
  }
  
  function addRing(axisString, horizAngle, vertAngle, radius=10, color=0x000000) {
    const ringObj = {axisString, horizAngle, vertAngle, radius, color}

    let circleGeometry;
    const circleMaterial = new THREE.MeshStandardMaterial({color: color});
    let circle;
    if (axisString == "x") {
      const ringRadius = radius * Math.sqrt(Math.cos(vertAngle)**2 + (Math.sin(horizAngle) * Math.sin(vertAngle))**2);
      circleGeometry = new THREE.TorusGeometry(ringRadius, 0.1, 3, 100, 2*Math.PI);
      circle = new THREE.Mesh(circleGeometry, circleMaterial);
      circle.translateX(radius * Math.cos(horizAngle) * Math.sin(vertAngle));
      circle.rotateY(Math.PI/2);
    }
    if (axisString == "y") {
      const ringRadius = radius * Math.sin(vertAngle);
      circleGeometry = new THREE.TorusGeometry(ringRadius, 0.1, 3, 100, 2*Math.PI);
      circle = new THREE.Mesh(circleGeometry, circleMaterial);
      circle.translateY(radius * Math.cos(vertAngle));
      circle.rotateX(Math.PI/2);
    }
    if (axisString == "z") {
      const ringRadius = radius * Math.sqrt(Math.cos(vertAngle)**2 + (Math.cos(horizAngle) * Math.sin(vertAngle))**2);
      circleGeometry = new THREE.TorusGeometry(ringRadius, 0.1, 3, 100, 2*Math.PI);
      circle = new THREE.Mesh(circleGeometry, circleMaterial);
      circle.translateZ(-radius * Math.sin(horizAngle) * Math.sin(vertAngle));
    }
    scene.add(circle);
    ringObj.ring = circle;

    return ringObj;
  }

  function addDot(horizAngle=-Math.PI/2, vertAngle, radius=10, color=0x000000, group, axis) {
    const dotObj = {horizAngle, vertAngle, radius, color, group, axis};

    const dotGeometry = new THREE.SphereGeometry(0.3, 8, 8);
    const dotMaterial = new THREE.MeshStandardMaterial({color: color});
    const dot = new THREE.Mesh(dotGeometry, dotMaterial);
    dot.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), (-vertAngle)%(2*Math.PI));
    dot.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
    dot.translateY(radius);
    scene.add(dot);
    dotObj.dot = dot;

    return dotObj;
  }

  function addGlobalPhaseDots(vertAngle, radius=10, color=0x000000) {
    const dotArr = [];

    dotArr.push(addDot(-Math.PI/2, (vertAngle + Math.PI)%(2*Math.PI), radius, color));
    dotArr.push(addDot(-Math.PI/2, (vertAngle)%(2*Math.PI), radius, color));

    return dotArr;
  }

  function addMeasurementDots(vertAngle, radius=10, color=0x000000, axis='Z') {
    const dotArr = [];

    let dot0;
    let dot1;
    let dot2;
    let dot3;

    if (axis == 'Z') {
      dot0 = addDot(-Math.PI/2, (vertAngle)%(2*Math.PI), radius, color);
      dot1 = addDot(-Math.PI/2, (-vertAngle + Math.PI + 2*Math.PI)%(2*Math.PI), radius, color);
      dot2 = addDot(-Math.PI/2, (vertAngle + Math.PI)%(2*Math.PI), radius, color);
      dot3 = addDot(-Math.PI/2, (-vertAngle + 2*Math.PI)%(2*Math.PI), radius, color);
    }
    if (axis == 'X') {
      dot0 = addDot(-Math.PI/2, (vertAngle)%(2*Math.PI), radius, color);
      dot1 = addDot(-Math.PI/2, (-vertAngle + 3*Math.PI + Math.PI/2)%(2*Math.PI), radius, color);
      dot2 = addDot(-Math.PI/2, (vertAngle + Math.PI)%(2*Math.PI), radius, color);
      dot3 = addDot(-Math.PI/2, (-vertAngle + 2*Math.PI + Math.PI/2)%(2*Math.PI), radius, color);
    }

    dot0.group = vertAngle;
    dot1.group = vertAngle;
    dot2.group = vertAngle;
    dot3.group = vertAngle;

    dot0.axis = axis;
    dot1.axis = axis;
    dot2.axis = axis;
    dot3.axis = axis;

    dotArr.push(dot0);
    dotArr.push(dot1);
    dotArr.push(dot2);
    dotArr.push(dot3);

    return dotArr;
  }

  function removeGroups(objArr) {
    for (let obj of objArr) {
      for (let key of Object.keys(obj)) {
        if (obj[key] && (obj[key].type == "Mesh" || obj[key].type == "Sprite")) {
          scene.remove(obj[key]);
        }
      }
    }
  }

  function rotateGroups(objArr, axisArr, angle) {
    const axis = new THREE.Vector3(axisArr[0], axisArr[1], axisArr[2]);
    for (let obj of objArr) {
      for (let key of Object.keys(obj)) {
        if (obj[key] && (obj[key].type == "Mesh" || obj[key].type == "Sprite")) {
          obj[key].applyMatrix4(new THREE.Matrix4().makeRotationAxis(axis, angle));
        }
      }
    }
  }

  function prepareStateToOutput(alphaExpr, betaExpr) {
    let newAlphaExpr = alphaExpr;
    let newBetaExpr = betaExpr;
    newAlphaExpr = math.format(math.evaluate(newAlphaExpr), 2).toString();
    newBetaExpr = math.format(math.evaluate(newBetaExpr), 2).toString();

    newAlphaExpr = newAlphaExpr.replaceAll(" ", "");
    newBetaExpr = newBetaExpr.replaceAll(" ", "");

    while (newAlphaExpr.length !== newBetaExpr.length) {
      if (newAlphaExpr.length > newBetaExpr.length) {
        newBetaExpr = " " + newBetaExpr;
      }
      if (newAlphaExpr.length > newBetaExpr.length) {
        newBetaExpr = newBetaExpr + " ";
      }

      if (newAlphaExpr.length < newBetaExpr.length) {
        newAlphaExpr = " " + newAlphaExpr;
      }
      if (newAlphaExpr.length < newBetaExpr.length) {
        newAlphaExpr = newAlphaExpr + " ";
      }
    }

    return [newAlphaExpr, newBetaExpr];
  }

  function toStandardStateForm(alphaExpr, betaExpr) {
    let alpha = math.evaluate(alphaExpr);
    let beta = math.evaluate(betaExpr);
    if (alpha.im && alpha.im != 0) {
      alpha = math.multiply(alpha, math.i);
      beta = math.multiply(beta, math.i);
    }

    if (alpha < 0) {
      alpha *= -1;
      beta *= -1;
    }

    let newAlphaExpr = alpha.toString();
    let newBetaExpr = beta.toString();

    return [newAlphaExpr, newBetaExpr];
  }

  function applyTransform(alphaExpr, betaExpr, matrix=[['1', '0'], ['0', '1']]) {
    const newAlphaNum = math.evaluate("(("+matrix[0][0]+')*('+alphaExpr+"))+(("+matrix[0][1]+')*('+betaExpr+"))");
    const newBetaNum = math.evaluate("(("+matrix[1][0]+')*('+alphaExpr+"))+(("+matrix[1][1]+')*('+betaExpr+"))");

    return [newAlphaNum.toString(), newBetaNum.toString()];
  }
  
  function toSinInterval(val) {
    if (val < -1) {
      val = -1;
    }
    if (val > 1) {
      val = 1;
    }
    return val;
  }

  function addStateToRealPlane(alphaExpr, betaExpr, axis='Z', color) {
    let arrowObj;
    let result;
    if (axis == 'Z') {
      result = applyTransform(alphaExpr, betaExpr);
    }
    if (axis == 'X') {
      result = applyTransform(alphaExpr, betaExpr,
        [["1/sqrt(2)", "1/sqrt(2)"],
          ["1/sqrt(2)", "-1/sqrt(2)"]]
      );
    }
    if (axis == 'Y') {
      result = applyTransform(alphaExpr, betaExpr,
        [["1/sqrt(2)", "1/sqrt(2)"],
          ["-i/sqrt(2)", "i/sqrt(2)"]]
      );
    }

    let alpha = math.evaluate(result[0]);
    if (alpha.im && alpha.im != 0){
      return undefined;
    }
    let beta = math.evaluate(result[1]);
    if (beta.im && beta.im != 0){
      return undefined;
    }

    const vertAngle = Math.atan2(toSinInterval(alpha), toSinInterval(beta))

    const toOutput = prepareStateToOutput(result[0], result[1]);

    arrowObj = addArrow(undefined, vertAngle, 5, color, 5, "⎧"+toOutput[0]+"⎫\n⎩"+toOutput[1]+"⎭");

    return arrowObj;
  }

  function addStateToBlochSphere(alphaExpr, betaExpr, axis='Z', color) {
    let arrowObj;
    let result;
    if (axis == 'Z') {
      result = applyTransform(alphaExpr, betaExpr);
    }
    if (axis == 'X') {
      result = applyTransform(alphaExpr, betaExpr,
        [["1/sqrt(2)", "1/sqrt(2)"],
          ["1/sqrt(2)", "-1/sqrt(2)"]]
      );
    }
    if (axis == 'Y') {
      result = applyTransform(alphaExpr, betaExpr,
        [["1/sqrt(2)", "1/sqrt(2)"],
          ["i/sqrt(2)", "-i/sqrt(2)"]]
      );
    }

    let alpha = math.evaluate(result[0]);
    let beta = math.evaluate(result[1]);

    if (alpha.im && alpha.im != 0){
      const phaseDelta = math.evaluate(`e^(i*(${math.atan2(alpha.im, alpha.re)}))`);
      alpha = math.evaluate(`(${alpha.toString()})/(${phaseDelta.toString()})`);
      beta = math.evaluate(`(${beta.toString()})/(${phaseDelta.toString()})`);
      alpha = alpha.re;
    }

    const vertAngle = 2*math.acos(toSinInterval(alpha));

    const betaSin = math.sin(vertAngle/2);
    let betaE;
    
    if (betaSin != 0) {
      betaE = math.evaluate("("+beta+")/("+betaSin+")");
    }
    else {
      betaE = 0;
    }

    let horizAngle;
    if (betaE.im && betaE.im != 0) {
      horizAngle = Math.atan2(toSinInterval(betaE.im), toSinInterval(betaE.re));
    }
    else {
      horizAngle = Math.atan2(0, toSinInterval(betaE));
    }

    const toOutput = prepareStateToOutput(alpha.toString(), beta.toString());

    arrowObj = addArrow((horizAngle - Math.PI/2 + 4*Math.PI)%(2*Math.PI), vertAngle, 10, color, 0, "⎧"+toOutput[0]+"⎫\n⎩"+toOutput[1]+"⎭");

    return arrowObj;
  }

  function initScene() {
    const ambientLight = new THREE.AmbientLight(0xFFFFF0);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2);
    directionalLight.position.set(-30, 50, 0);
    directionalLight.castShadow = true;
    scene.add(directionalLight);

    curStateObj.main = addRealPLane();
  }

  function addMeasurementToBlochSphere(curStateObj, axis='Z', horizAngleIn, vertAngle, radius=10, color=0x000000) {
    const horizAngle = horizAngleIn+Math.PI/2;
    let curMeasurObj = curStateObj.curMeasurObj;
    if (curMeasurObj == undefined || !curStateObj.finished) {
      if (curMeasurObj == undefined) {
        curMeasurObj = {axis, horizAngle, vertAngle, radius, color};
        curStateObj.curMeasurObj = curMeasurObj;

        curStateObj.finished = false;

        const alphaExpr = math.evaluate(`cos(${vertAngle}/2)`).toString();
        const betaExpr = math.evaluate(`e^(i*${horizAngle})*sin(${vertAngle}/2)`).toString();

        let result;
        if (axis == 'Z') {
          result = applyTransform(alphaExpr, betaExpr);
        }
        if (axis == 'X') {
          result = applyTransform(alphaExpr, betaExpr,
            [["1/sqrt(2)", "1/sqrt(2)"],
            ["1/sqrt(2)", "-1/sqrt(2)"]]
          );
        }
        if (axis == 'Y') {
          result = applyTransform(alphaExpr, betaExpr,
            [["1/sqrt(2)", "-i/sqrt(2)"],
            ["1/sqrt(2)", "i/sqrt(2)"]]
          );
        }

        let arrowVertAngle;
        let arrowHorizAngle;

        let viewPortAxis;
        if (axis == 'Z') {
          viewPortAxis = 'y';
          curMeasurObj.finalArrowRadius = radius*(1 + math.cos(vertAngle));
          arrowVertAngle = 0;
          arrowHorizAngle = 0;
        }
        if (axis == 'X') {
          viewPortAxis = 'z';
          curMeasurObj.finalArrowRadius = radius*(1 + math.sin(vertAngle)*math.cos(horizAngle));
          arrowVertAngle = Math.PI/2;
          arrowHorizAngle = 3*Math.PI/2;
        }
        if (axis == 'Y') {
          viewPortAxis = 'x';
          curMeasurObj.finalArrowRadius = radius*(1 - math.sin(vertAngle)*math.sin(horizAngle));
          arrowVertAngle = Math.PI/2;
          arrowHorizAngle = Math.PI;
        }

        const meas = prepareStateToOutput(math.evaluate(`(${result[0]})*conj(${result[0]})`).toString(),
          math.evaluate(`(${result[1]})*conj(${result[1]})`).toString());

        const temp = math.max(math.evaluate(meas[0]), math.evaluate(meas[1])).toString();
        meas[0] = math.min(math.evaluate(meas[0]), math.evaluate(meas[1])).toString();
        meas[1] = temp;

        if (curMeasurObj.finalArrowRadius > (radius - curMeasurObj.finalArrowRadius/2)) {
          const temp2 = meas[0];
          meas[0] = meas[1];
          meas[1] = temp2;
        }

        curMeasurObj.ring = addRing(viewPortAxis, horizAngleIn, vertAngle, radius, color);
        curMeasurObj.arrow1 = addArrow(arrowHorizAngle, arrowVertAngle, 0, color, -radius, meas[0], true);
        curMeasurObj.arrow2 = addArrow(arrowHorizAngle, arrowVertAngle + Math.PI, 0, color, -radius, meas[1], true);
        curMeasurObj.step = 0;
      }
      else {
        if (curMeasurObj.step > 180) {
          curMeasurObj.finished = true;
          inputObj.addMeasurementAnimation = false;
        }
        else {
          curMeasurObj.step += 1;
          const prevArrow1 = curMeasurObj.arrow1;
          const prevArrow2 = curMeasurObj.arrow2;
          removeGroups([curMeasurObj.arrow1, curMeasurObj.arrow2]);
          curMeasurObj.ring.ring.scale.y -= 1/180;
          curMeasurObj.ring.ring.scale.x -= 1/180;
          const newRadius1 = prevArrow1.radius + curMeasurObj.finalArrowRadius/360;
          const newRadius2 = prevArrow2.radius + (radius - curMeasurObj.finalArrowRadius/2)/180;
          curMeasurObj.arrow1 = addArrow(prevArrow1.horizAngle, prevArrow1.vertAngle, newRadius1, color, (newRadius1 - radius), prevArrow1.labelText, true);
          curMeasurObj.arrow2 = addArrow(prevArrow2.horizAngle, prevArrow2.vertAngle, newRadius2, color, (newRadius2 - radius), prevArrow2.labelText, true);
        }
      }
    }
    return curStateObj;
  }

  function addQubitParticle(curStateObj, up=true, radius=10, appear=true, color=0x0000ff) {
    if (curStateObj.qubitParticle == undefined || !curStateObj.finished) {
      if (curStateObj.qubitParticle == undefined) {
        const qubitParticle = {};
        curStateObj.qubitParticle = qubitParticle;
        const sphereGeometry = new THREE.SphereGeometry(1.2*radius, 100, 100);
        const sphereMaterial = new THREE.MeshStandardMaterial({color: color, transparent: true});
        sphereMaterial.opacity = 0;
        const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial);
        scene.add(sphere);
        qubitParticle.sphere = sphere;

        const circleGeometryXZ = new THREE.TorusGeometry(1.5*radius, 0.5, 50, 100, Math.PI);
        const circleMaterialXZ = new THREE.MeshStandardMaterial({color: color, transparent: true});
        circleMaterialXZ.opacity = 0;
        const circleXZ = new THREE.Mesh(circleGeometryXZ, circleMaterialXZ);
        circleXZ.rotateX(Math.PI/2);
        circleXZ.rotateZ(3.5*Math.PI/2);
        scene.add(circleXZ);
        qubitParticle.circleXZ = circleXZ;

        const arrowGeometry = new THREE.ConeGeometry(1.5, 0.3*radius, 50, 1);
        const arrowMaterial = new THREE.MeshStandardMaterial({color: color, transparent: true});
        arrowMaterial.opacity = 0;
        const arrow = new THREE.Mesh(arrowGeometry, arrowMaterial);
        arrow.translateZ(-1.5*radius/Math.sqrt(2));
        arrow.translateX(1.5*radius/Math.sqrt(2));
        arrow.rotateX(Math.PI/2);
        arrow.rotateZ(1.5*Math.PI/2);
        scene.add(arrow);
        qubitParticle.arrow = arrow;

        const lineGeometryY = new THREE.CylinderGeometry(0.5, 0.5, 2.9*radius, 50, 1);
        const lineMaterialY = new THREE.MeshStandardMaterial({color: color, transparent: true});
        lineMaterialY.opacity = 0;
        const lineY = new THREE.Mesh(lineGeometryY, lineMaterialY);
        scene.add(lineY);
        qubitParticle.lineY = lineY;

        const arrowGeometryY0 = new THREE.ConeGeometry(1.5, 0.3*radius, 50, 1);
        const arrowMaterialY0 = new THREE.MeshStandardMaterial({color: color, transparent: true});
        arrowMaterialY0.opacity = 0;
        const arrowY0 = new THREE.Mesh(arrowGeometryY0, arrowMaterialY0);
        arrowY0.translateY(1.6*radius);
        scene.add(arrowY0);
        qubitParticle.arrowY0 = arrowY0;

        if (!up) {
          rotateGroups([qubitParticle], [0, 0, 1], Math.PI);
        }

        curStateObj.finished = false;
      }
      else {
        const qubitParticle = curStateObj.qubitParticle;
        if ((appear && qubitParticle.sphere.material.opacity < 0.5) || (!appear && qubitParticle.sphere.material.opacity > 0.5)) {
          if (appear) {
            qubitParticle.sphere.material.opacity += 1/180;
            qubitParticle.circleXZ.material.opacity += 1/180;
            qubitParticle.arrow.material.opacity += 1/180;
            qubitParticle.lineY.material.opacity += 1/180;
            qubitParticle.arrowY0.material.opacity += 1/180;
          }
          else {
            qubitParticle.sphere.material.opacity -= 1/180;
            qubitParticle.circleXZ.material.opacity -= 1/180;
            qubitParticle.arrow.material.opacity -= 1/180;
            qubitParticle.lineY.material.opacity -= 1/180;
            qubitParticle.arrowY0.material.opacity -= 1/180;
          }
        }
        else {
          curStateObj.finished = true;
          inputObj.addQubitParticleAnimation = false;
        }
      }
    }
    return curStateObj;
  }

  function realPlaneToBlochSphere(curStateObj) {
    const main = curStateObj.main;
    if (main.circleXY == undefined || !curStateObj.finished) {
      if (main.circleXY == undefined) {
        if (main.arrowY1.position.y > -11.999) {
          rotateGroups([main].concat(curStateObj.arrowObjArr).concat(curStateObj.dotObjArr), [0, 0, 1], Math.PI/180);
        } 
        else {
          const circleGeometryXY = new THREE.TorusGeometry(10, 0.1, 3, 100, 2*Math.PI);
          const circleMaterialXY = new THREE.MeshStandardMaterial({color: 0x000000, transparent: true});
          const circleXY = new THREE.Mesh(circleGeometryXY, circleMaterialXY);
          circleXY.material.opacity = 0;
          scene.add(circleXY);
          main.circleXY = circleXY;

          const circleGeometryXZ = new THREE.TorusGeometry(10, 0.1, 3, 100, 2*Math.PI);
          const circleMaterialXZ = new THREE.MeshStandardMaterial({color: 0x000000, transparent: true});
          const circleXZ = new THREE.Mesh(circleGeometryXZ, circleMaterialXZ);
          circleXZ.material.opacity = 0;
          circleXZ.rotateX(Math.PI/2);
          scene.add(circleXZ);
          main.circleXZ = circleXZ;

          const lineGeometryX = new THREE.CylinderGeometry(0.1, 0.1, 24, 3, 1);
          const lineMaterialX = new THREE.MeshStandardMaterial({color: 0x000000, transparent: true});
          const lineX = new THREE.Mesh(lineGeometryX, lineMaterialX);
          lineX.material.opacity = 0;
          lineX.rotateZ(Math.PI/2);
          scene.add(lineX);
          main.lineX = lineX;

          const arrowGeometryX0 = new THREE.ConeGeometry(0.4, 1, 10, 1);
          const arrowMaterialX0 = new THREE.MeshStandardMaterial({color: 0x000000, transparent: true});
          const arrowX0 = new THREE.Mesh(arrowGeometryX0, arrowMaterialX0);
          arrowX0.material.opacity = 0;
          arrowX0.translateX(12);
          arrowX0.rotateZ(-Math.PI/2);
          scene.add(arrowX0);
          main.arrowX0 = arrowX0;

          const arrowGeometryX1 = new THREE.ConeGeometry(0.4, 1, 10, 1);
          const arrowMaterialX1 = new THREE.MeshStandardMaterial({color: 0x000000, transparent: true});
          const arrowX1 = new THREE.Mesh(arrowGeometryX1, arrowMaterialX1);
          arrowX1.material.opacity = 0;
          arrowX1.translateX(-12);
          arrowX1.rotateZ(Math.PI/2);
          scene.add(arrowX1);
          main.arrowX1 = arrowX1;

          let labelX0 = new TextSprite({
            text: '|𝑖⟩',
            fontSize: 2,
            color: '#000000',
          });
          labelX0.material.opacity = 0;
          labelX0.translateX(14);
          scene.add(labelX0);
          main.labelX0 = labelX0;
          
          let labelX1 = new TextSprite({
              text: '|-𝑖⟩',
              fontSize: 2,
              color: '#000000',
          });
          labelX1.translateX(-14);
          labelX1.material.opacity = 0;
          scene.add(labelX1);
          main.labelX1 = labelX1;

          for (let i = 0; i < curStateObj.arrowObjArr.length; i++) {
            const prevObj = curStateObj.arrowObjArr[i];
            arrowVertAngleArr.push(prevObj.vertAngle);
            removeGroups([prevObj]);
            curStateObj.arrowObjArr[i] = addArrow(undefined, (-prevObj.vertAngle + Math.PI) % (2*Math.PI), prevObj.radius, prevObj.color, prevObj.fromCenter, prevObj.labelText);
          }
          for (let i = 0; i < curStateObj.dotObjArr.length; i++) {
            const prevObj = curStateObj.dotObjArr[i];
            dotsVertAngleArr.push(prevObj.vertAngle);
            removeGroups([prevObj]);
            curStateObj.dotObjArr[i] = addDot(undefined, (-prevObj.vertAngle + Math.PI) % (2*Math.PI), prevObj.radius, prevObj.color, prevObj.group, prevObj.axis);
          }
        }
      }
      else {
        if (main.circleXY.material.opacity < 0.99) {
          main.circleXY.material.opacity += 1/180;
          main.circleXZ.material.opacity += 1/180;
          main.lineX.material.opacity += 1/180;
          main.arrowX0.material.opacity += 1/180;
          main.arrowX1.material.opacity += 1/180;
          main.labelX0.material.opacity += 1/180;
          main.labelX1.material.opacity += 1/180;

          for (let i = 0; i < curStateObj.arrowObjArr.length; i++) {
            const prevObj = curStateObj.arrowObjArr[i];
            removeGroups([curStateObj.arrowObjArr[i]]);
            curStateObj.arrowObjArr[i] = addArrow(undefined, prevObj.vertAngle-arrowVertAngleArr[i]/180, prevObj.radius + 5/180, prevObj.color, prevObj.fromCenter - 5/180, prevObj.labelText);
          }
          for (let i = 0; i < curStateObj.dotObjArr.length; i++) {
            const prevObj = curStateObj.dotObjArr[i];
            removeGroups([curStateObj.dotObjArr[i]]);
            curStateObj.dotObjArr[i] = addDot(undefined, prevObj.vertAngle-dotsVertAngleArr[i]/180, prevObj.radius, prevObj.color, prevObj.group, prevObj.axis);
          }

          rotateGroups([{arrowY0: main.arrowY0}], [1, 0, 0], -Math.PI/2/180);
          rotateGroups([{lineY0: main.lineY0}], [1, 0, 0], -Math.PI/2/180);
          rotateGroups([{lineZ0: main.lineZ0}], [1, 0, 0], -Math.PI/4/180);
          rotateGroups([{arrowZ0: main.arrowZ0}], [1, 0, 0], -Math.PI/4/180);
          rotateGroups([{lineZ1: main.lineZ1}], [1, 0, 0], -3*Math.PI/4/180);
          rotateGroups([{arrowZ1: main.arrowZ1}], [1, 0, 0], -3*Math.PI/4/180);
          rotateGroups([{labelY0: main.labelY0}], [1, 0, 0], -Math.PI/2/180);
          rotateGroups([{labelZ0: main.labelZ0}], [1, 0, 0], -Math.PI/4/180);
          rotateGroups([{labelZ1: main.labelZ1}], [1, 0, 0], -3*Math.PI/4/180);
        }
        else {
          removeGroups([curStateObj.main]);
          curStateObj.main = addBlochSphere();

          for (let i = 0; i < curStateObj.arrowObjArr.length; i++) {
            const prevObj = curStateObj.arrowObjArr[i];
            removeGroups([curStateObj.arrowObjArr[i]]);
            curStateObj.arrowObjArr[i] = addArrow(undefined, (Math.PI - 2*arrowVertAngleArr[i])%(2*Math.PI), 10, prevObj.color, 0, prevObj.labelText);
          }

          const dotGroups = {};
          for (let dotObj of curStateObj.dotObjArr) {
            if (dotObj.group && !dotGroups[dotObj.group]) {
              let ringAxis;
              if (dotObj.axis == 'Z') {
                ringAxis = 'y';
              }
              if (dotObj.axis == 'X') {
                ringAxis = 'z';
              }
              const ring = addRing(ringAxis, -Math.PI/2, dotObj.vertAngle, undefined, dotObj.color);
              curStateObj.ring = ring;
              dotGroups[dotObj.group] = true;
            }
          }
          removeGroups(curStateObj.dotObjArr);

          arrowVertAngleArr = [];
          dotsVertAngleArr = [];

          curStateObj.finished = true;
          inputObj.toBlochSphereAnimation = false;
        }
      }
    }
    return curStateObj;
  }

  let renderer;

onDestroy( () => {
    if (renderer != undefined) {
        renderer.dispose();
        renderer.forceContextLoss();
    }
});

afterUpdate(async () => {
    if (document.getElementById("renderer")) {
        return;
    }
    let width = window.innerWidth;
    let height = window.innerHeight;
    if (renderer != undefined) {
        renderer.dispose();
        renderer.forceContextLoss();
    }
    renderer = new THREE.WebGL1Renderer({antialias: true, alpha: true});
    renderer.autoClear = false;
    renderer.domElement.id = "renderer";
    scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, width / height, 0.01, 10000000);
    const orbit = new OrbitControls(camera, renderer.domElement);
    const mousePosition = new THREE.Vector2();
    renderer.setSize(width, height);
    document.getElementById("bloch-sphere-container").appendChild(renderer.domElement);
    camera.position.set(10, 10, 30);
    orbit.update();

    const backgroundScene = new THREE.Scene();
    async function initBackgroundScene() {
        const spaceModel = await loadModel(spaceUrl, "backgroundSpace", backgroundScene);
        spaceModel.translateX(-1.44*1000);
        spaceModel.translateY(-1.44*1000);
        spaceModel.translateZ(1.44*1000);
        spaceModel.scale.set(1000, 1000, 1000);

        const texture = cubetextureloader.load([
            backgroundNebulaNX,
            backgroundNebulaPX,
            backgroundNebulaPY,
            backgroundNebulaNY,
            backgroundNebulaNZ,
            backgroundNebulaPZ,
        ]);
        backgroundScene.background = texture;
    }
    initBackgroundScene();

    initScene();

    window.addEventListener('resize', () => {
      width = window.innerWidth;
      height = window.innerHeight;
      camera.aspect = width/height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
    });

    window.addEventListener('mousemove', function(e) {
        mousePosition.x = (e.clientX/window.innerWidth) * 2 - 1;
        mousePosition.y = - (e.clientY/window.innerHeight) * 2 + 1;
    });

    function animate(time) {
      if (inputObj.toBlochSphereAnimation) {
        curStateObj = realPlaneToBlochSphere(curStateObj);
      }
      if (inputObj.addMeasurementAnimation) {
        const arrow = curStateObj.arrowObjArr[0];
        curStateObj = addMeasurementToBlochSphere(curStateObj, inputObj.measurementBasis, arrow.horizAngle, arrow.vertAngle, arrow.radius, arrow.color);
      }
      if (inputObj.addQubitParticleAnimation) {
        curStateObj = addQubitParticle(curStateObj, inputObj.qubitParticle);
      }
      renderer.clear();
      renderer.render(backgroundScene, camera);
      renderer.render(scene, camera);
    }

    renderer.setAnimationLoop(animate);
  });
</script>

<div id="bloch-sphere-container">
  <div class="controls-container">
    <button on:click={reset}>Reset</button>
    <div class="matrix-grid-container">
      {#each Array.from(Array(2).keys()) as col}
        <input class="state" id="state-{col}" bind:value={inputObj.state[`${col}`]}/>
      {/each}
    </div>
    <button on:click={() => addStateClicked('Z')} disabled={curStateObj.arrowObjArr.length >= 1 || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add State in Z</button>
    <button on:click={() => addStateClicked('X')} disabled={curStateObj.arrowObjArr.length >= 1 || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add State in X</button>
    <button on:click={() => addStateClicked('Y')} disabled={curStateObj.arrowObjArr.length >= 1 || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add State in Y</button>
    <button on:click={toBlochSphereClicked} disabled={(curStateObj.main != undefined && curStateObj.main.circleXY != undefined) || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>To Bloch Sphere</button>
    <button on:click={() => addMeasurementDotsClicked('Z')} disabled={curStateObj.arrowObjArr.length < 1 || (curStateObj.main != undefined && curStateObj.main.circleXY != undefined) || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add Measurements Dots in Z</button>
    <button on:click={() => addMeasurementDotsClicked('X')} disabled={curStateObj.arrowObjArr.length < 1 || (curStateObj.main != undefined && curStateObj.main.circleXY != undefined) || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add Measurements Dots in X</button>
    <button on:click={addPhaseDotsClicked} disabled={(curStateObj.main != undefined && curStateObj.main.circleXY != undefined) || curStateObj.arrowObjArr.length < 1 || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add Global Phase Dots</button>
    <button on:click={() => addMeasurementClicked('Z')} disabled={curStateObj.curMeasurObj != undefined || curStateObj.arrowObjArr.length < 1 || (curStateObj.main == undefined || curStateObj.main.circleXY == undefined) || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Measure in Z</button>
    <button on:click={() => addMeasurementClicked('X')} disabled={curStateObj.curMeasurObj != undefined || curStateObj.arrowObjArr.length < 1 || (curStateObj.main == undefined || curStateObj.main.circleXY == undefined) || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Measure in X</button>
    <button on:click={() => addMeasurementClicked('Y')} disabled={curStateObj.curMeasurObj != undefined || curStateObj.arrowObjArr.length < 1 || (curStateObj.main == undefined || curStateObj.main.circleXY == undefined) || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Measure in Y</button>
    <button on:click={() => addQubitParticleClicked(true)} disabled={curStateObj.qubitParticle != undefined || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add qubit particle |0⟩</button>
    <button on:click={() => addQubitParticleClicked(false)} disabled={curStateObj.qubitParticle != undefined || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Add qubit particle |1⟩</button>
    <div class="matrix-grid-container">
      {#each Array.from(Array(2).keys()) as row}
          {#each Array.from(Array(2).keys()) as col}
              <input class="gate-matrix" id="gate-matrix-{row}{col}" bind:value={inputObj.transformMatrix[`${row}-${col}`]}/>
          {/each}
      {/each}
    </div>
    <button on:click={applyGateClicked} disabled={curStateObj.arrowObjArr.length < 1 || curStateObj.curMeasurObj != undefined || inputObj.toBlochSphereAnimation || inputObj.addMeasurementAnimation || inputObj.addQubitParticleAnimation}>Apply Gate</button>
  </div>
</div>

<style>
  .controls-container {
    position: absolute;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .matrix-grid-container {
      display: grid;
      grid-template-columns: repeat(2, min(10vh, 5vw));
  }
</style>