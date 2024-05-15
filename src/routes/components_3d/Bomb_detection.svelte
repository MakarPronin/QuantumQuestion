<script>
    import { afterUpdate, onDestroy } from "svelte";
    import * as THREE from "three";
    import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
    import TextSprite from "@seregpie/three.text-sprite";
    import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
    
    import pdfUrlRotationGate from '../assets/images/RotationGate.png?url';

    const droneUrl = new URL("../assets/models/surveillance_drone.glb", import.meta.url);
    const doorUrl = new URL("../assets/models/door.glb", import.meta.url);
    const bombUrl = new URL("../assets/models/round_bomb.glb", import.meta.url);
    const explosionUrl = new URL("../assets/models/explosion.glb", import.meta.url);
    const outsideDoorUrl = new URL("../assets/models/outside_door.glb", import.meta.url);

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

    let importantValuesObj = {rotationAngle: 0.1, quantumState0: [1, 0], quantumState1: [1, 0], needToRotate: false,
        needToRotateWithAnim: false, needsReset: false, measure: false};

    function measureClicked() {
        importantValuesObj.measure = true;
    }

    function resetClicked() {
        importantValuesObj.needsReset = true;
    }

    function rotationAngleUpdate(value) {
        importantValuesObj.rotationAngle = value;
    }

    function rotateClicked() {
        importantValuesObj.needToRotate = true;
    }

    function rotateWithAnimationClicked() {
        importantValuesObj.needToRotateWithAnim = true;
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
        const scene = new THREE.Scene();
        const mousePosition = new THREE.Vector2();
        renderer.setSize(width, height);
        renderer.shadowMap.enabled = true;
        document.getElementById("bomb-detection-container").appendChild(renderer.domElement);

        let camera = new THREE.PerspectiveCamera(75, width / height, 0.01, 10000000);
        const orbit = new OrbitControls(camera, renderer.domElement);
        camera.position.set(0, 0, 40);
        orbit.update();
        let labelPsi0;
        let labelPsi1;

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


        function initScene() {
            const ambientLight = new THREE.AmbientLight(0xFFFFF0);
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 4);
            directionalLight.position.set(-30, 30, 0);
            directionalLight.shadow.mapSize = new THREE.Vector2(512, 512);
            directionalLight.shadow.camera.scale.set(2, 8, 0.2);
            directionalLight.shadow.radius = 1.5;
            directionalLight.shadow.bias = -0.004;
            directionalLight.castShadow = true;
            scene.add(directionalLight);

            let labelQuantum = new TextSprite({
                text: 'With a Bomb',
                fontSize: 2,
                color: '#000000',
            });
            labelQuantum.translateX(-14);
            labelQuantum.translateY(25);
            scene.add(labelQuantum);

            let labelClassical = new TextSprite({
                text: 'Without a Bomb',
                fontSize: 2,
                color: '#000000',
            });
            labelClassical.translateX(14);
            labelClassical.translateY(25);
            scene.add(labelClassical);

            labelPsi0 = new TextSprite({
                text: `
     amplitudes        probabilities
     ⬇                  ⬇    
|ψ⟩ = ⎧${stateStringOutput(importantValuesObj.quantumState0[0])}⎫ |⟶⟩   ⟹  ⎧${stateStringOutput(importantValuesObj.quantumState0[0]**2)}⎫ |⟶⟩
     ⎩${stateStringOutput(importantValuesObj.quantumState0[1])}⎭ |↘⟩        ⎩${stateStringOutput(importantValuesObj.quantumState0[1]**2)}⎭ |↘⟩
            `,
                fontSize: 1,
                color: '#000000',
                fontFamily: "monospace"
            });
            labelPsi0.translateX(-15);
            labelPsi0.translateY(20);
            scene.add(labelPsi0);

            labelPsi1 = new TextSprite({
                text: `
     amplitudes        probabilities
     ⬇                  ⬇    
|ψ⟩ = ⎧${stateStringOutput(importantValuesObj.quantumState1[0])}⎫ |⟶⟩   ⟹  ⎧${stateStringOutput(importantValuesObj.quantumState1[0]**2)}⎫ |⟶⟩
     ⎩${stateStringOutput(importantValuesObj.quantumState1[1])}⎭ |↘⟩        ⎩${stateStringOutput(importantValuesObj.quantumState1[1]**2)}⎭ |↘⟩
            `,
                fontSize: 1,
                color: '#000000',
                fontFamily: "monospace"
            });
            labelPsi1.translateX(15);
            labelPsi1.translateY(20);
            scene.add(labelPsi1);

            const centralLineGeometry = new THREE.CylinderGeometry(0.1, 0.1, 60, 3, 1);
            const centralLineMaterial = new THREE.MeshStandardMaterial({color: 0x000000});
            const centralLine = new THREE.Mesh(centralLineGeometry, centralLineMaterial);
            scene.add(centralLine);
        }

        function createSquareFrame(pos={x: 0, y: 0, z: 0}, outerSize=18, thickness=1) {
            let shape = new THREE.Shape();
            shape.moveTo(-outerSize / 2, -outerSize / 2);
            shape.lineTo(outerSize / 2, -outerSize / 2);
            shape.lineTo(outerSize / 2, outerSize / 2);
            shape.lineTo(-outerSize / 2, outerSize / 2);
            shape.lineTo(-outerSize / 2, -outerSize / 2);

            let hole = new THREE.Path();
            let innerSize = outerSize - 2 * thickness;
            hole.moveTo(-innerSize / 2, -innerSize / 2);
            hole.lineTo(innerSize / 2, -innerSize / 2);
            hole.lineTo(innerSize / 2, innerSize / 2);
            hole.lineTo(-innerSize / 2, innerSize / 2);
            hole.lineTo(-innerSize / 2, -innerSize / 2);
            shape.holes.push(hole);

            let geometry = new THREE.ShapeGeometry(shape);
            let material = new THREE.MeshBasicMaterial({ color: 0x00ff00, side: THREE.DoubleSide});
            let squareFrame = new THREE.Mesh(geometry, material);

            squareFrame.position.set(pos.x, pos.y, pos.z);
            scene.add(squareFrame);

            return squareFrame;
        }

        function randomSquareFrame(side=false, alphaProbability) {

            const pos00 = {x: -15, y: -15, z: 0};
            const pos10 = {x: 15, y: -15, z: 0};
            const pos01 = {x: -35, y: -15, z: 0};
            const pos11 = {x: 35, y: -15, z: 0};

            if (!side) {
                if (Math.random() <= alphaProbability) {
                    return [createSquareFrame(pos00), false];
                }
                else {
                    return [createSquareFrame(pos01), true];
                }
            }
            else {
                if (Math.random() <= alphaProbability) {
                    return [createSquareFrame(pos10), false];
                }
                else {
                    return [createSquareFrame(pos11), true];
                }
            }
        }

        function updateStateNumbers(quantumState0, quantumState1) {
            labelPsi0.text = `
     amplitudes        probabilities
     ⬇                  ⬇    
|ψ⟩ = ⎧${stateStringOutput(quantumState0[0])}⎫ |⟶⟩   ⟹  ⎧${stateStringOutput(quantumState0[0]**2)}⎫ |⟶⟩
     ⎩${stateStringOutput(quantumState0[1])}⎭ |↘⟩        ⎩${stateStringOutput(quantumState0[1]**2)}⎭ |↘⟩
            `;

            labelPsi1.text = `
     amplitudes        probabilities
     ⬇                  ⬇    
|ψ⟩ = ⎧${stateStringOutput(quantumState1[0])}⎫ |⟶⟩   ⟹  ⎧${stateStringOutput(quantumState1[0]**2)}⎫ |⟶⟩
     ⎩${stateStringOutput(quantumState1[1])}⎭ |↘⟩        ⎩${stateStringOutput(quantumState1[1]**2)}⎭ |↘⟩
            `;
        }

        function rotateState(state, rotationAngle) {
            let alpha = state[0]*Math.cos(rotationAngle) - state[1]*Math.sin(rotationAngle);
            let beta = state[0]*Math.sin(rotationAngle) + state[1]*Math.cos(rotationAngle);

            if (alpha > 1 || beta == 0) {
                alpha = 1;
            }
            if (alpha < 0 || beta == 1) {
                alpha = 0;
            }

            if (beta > 1 || alpha == 0) {
                beta = 1;
            }
            if (beta < 0 || alpha == 1) {
                beta = 0;
            }

            return [alpha, beta];
        }

        function stateStringOutput(val) {
            let valStr = Math.abs(Math.round(val*100)/100).toPrecision(3).toString();
            valStr = valStr.substring(0, 4);

            return valStr;
        }

        function removeGroups(objArr) {
            for (let obj of objArr) {
                if (obj) {
                    for (let key of Object.keys(obj)) {
                        if (key == "droneObj" || key == "droneObj2") {
                            for (let inKey of Object.keys(obj[key])) {
                                scene.remove(obj[key][inKey]);
                            }
                        }
                        else if (obj[key] && obj[key].isObject3D) {
                            scene.remove(obj[key]);
                        }
                    }
                }
            }
        }

        function translateGroups(objArr, dir={x: 0,y: 0,z: 0}) {
            for (let obj of objArr) {
                if (obj) {
                    for (let key of Object.keys(obj)) {
                        if (key == "droneObj" || key == "droneObj2") {
                            for (let inKey of Object.keys(obj[key])) {
                                obj[key][inKey].applyMatrix4(new THREE.Matrix4().makeTranslation(dir.x, dir.y, dir.z))
                            }
                        }
                        else if (obj[key] && obj[key].isObject3D) {
                            obj[key].applyMatrix4(new THREE.Matrix4().makeTranslation(dir.x, dir.y, dir.z))
                        }
                    }
                }
            }
        }

        function changeLightHelperMaterial(lightHelper, opacity) {
            lightHelper.children[0].material = new THREE.MeshStandardMaterial({color: 0x000000, transparent: true, opacity});
            return lightHelper;
        }

        function changeOpacity(obj3D, opacity) {
            for (let key of Object.keys(obj3D)) {
                if (key == "droneObj" || key == "droneObj2") {
                    for (let inKey of Object.keys(obj3D[key])) {
                        obj3D[key][inKey].traverse((element) => {
                            if (element.isObject3D && element.material) {
                                element.material.transparent = true;
                                element.material.opacity = opacity;
                            }
                        });
                    }
                }
                else if (obj3D[key] && obj3D[key].isObject3D) {
                    obj3D[key].traverse((element) => {
                        if (element.isObject3D && element.material) {
                            element.material.transparent = true;
                            if (element.name.substring(0, 6) == "window") {
                                element.material.opacity = 0.25;
                            }
                            else {
                                element.material.opacity = opacity;
                            }
                        }
                    });
                }
            }
        }

        function changeVisibility(obj3D, visible) {
            for (let key of Object.keys(obj3D)) {
                if (key == "droneObj" || key == "droneObj2") {
                    for (let inKey of Object.keys(obj3D[key])) {
                        obj3D[key][inKey].traverse((element) => {
                            if (element.isObject3D) {
                                element.visible = visible;
                            }
                        });
                    }
                }
                else if (obj3D[key] && obj3D[key].isObject3D) {
                    obj3D[key].traverse((element) => {
                        if (element.isObject3D) {
                            element.visible = visible;
                        }
                    });
                }
            }
        }

        function rotateDrone(bombSceneObj, bottom) {
            const droneObj = bombSceneObj.droneObj;
            if (bottom) {
                droneObj.drone.rotateX(-Math.PI/4);
            }

            if (bottom) {
                droneObj.droneSpotLight.position.set(droneObj.drone.position.x + 0.6, droneObj.drone.position.y-0.5, droneObj.drone.position.z);
            }
            else {
                droneObj.droneSpotLight.position.set(droneObj.drone.position.x + 0.6, droneObj.drone.position.y, droneObj.drone.position.z);
            }

            if (bottom) {
                droneObj.droneSpotLight.target.position.set(droneObj.drone.position.x + 2, droneObj.drone.position.y-2, droneObj.drone.position.z);
            }
            else {
                droneObj.droneSpotLight.target.position.set(droneObj.drone.position.x + 2, droneObj.drone.position.y-0.5, droneObj.drone.position.z);
            }

            scene.remove(droneObj.dronePointLightHelper);
            const dronePointLightHelper = new THREE.SpotLightHelper(droneObj.droneSpotLight, 0x000000);
            scene.add(dronePointLightHelper);
            droneObj.dronePointLightHelper = dronePointLightHelper;

            bombSceneObj.bottom = bottom;

            bombSceneObj.droneObj = droneObj;

            return bombSceneObj;
        }

        async function addDrone(id="", pos={x: 0, y: 0, z: 0}, bottom=false, transparent=false){
            const droneObj = {};

            const drone = await loadModel(droneUrl, "drone" + id, scene);
            drone.scale.set(1.5, 1.5, 1.5);
            drone.rotateY(Math.PI/2);
            drone.position.set(pos.x, pos.y, pos.z);
            drone.translateZ(-2.5);
            drone.translateY(4);
            if (bottom) {
                drone.rotateX(Math.PI/4);
            }
            droneObj.drone = drone;
            if (transparent) {
                drone.traverse((element) => {
                    if (element.material) {
                        element.material.transparent = true;
                        element.material.opacity = 0;
                    }
                });
            }

            const droneSpotLight = new THREE.SpotLight(0xFFaaaa, 100000, 13, Math.PI/8, 1.0);
            droneSpotLight.shadow.mapSize = new THREE.Vector2(128, 128);
            droneSpotLight.shadow.bias = -0.003;
            if (bottom) {
                droneSpotLight.position.set(drone.position.x + 0.6, drone.position.y-0.5, drone.position.z);
            }
            else {
                droneSpotLight.position.set(drone.position.x + 0.6, drone.position.y, drone.position.z);
            }
            droneSpotLight.castShadow = true;
            scene.add(droneSpotLight);
            droneObj.droneSpotLight = droneSpotLight;

            if (bottom) {
                droneSpotLight.target.position.set(drone.position.x + 2, drone.position.y-2, drone.position.z);
            }
            else {
                droneSpotLight.target.position.set(drone.position.x + 2, drone.position.y-0.5, drone.position.z);
            }
            scene.add( droneSpotLight.target );
            droneObj.droneSpotLightTarget = droneSpotLight.target;

            if (!transparent) {
                const dronePointLightHelper = new THREE.SpotLightHelper(droneSpotLight, 0x000000);
                scene.add(dronePointLightHelper);
                droneObj.dronePointLightHelper = dronePointLightHelper;
            }

            return droneObj;
        }

        async function addBombScene(id="", pos={x: 0, y: 0, z: 0}, bottom=false, bombExists=true, transparent=false) {

            let bombSceneObj = {id, pos, bottom, bombExists};

            const door = await loadModel(doorUrl, "door" + id, scene);
            const window = scene.getObjectByName("Plane008_������001_0");
            window.name = "window" + id //allows to update castShadow in other copies.
            window.castShadow = false;
            door.scale.set(1.5, 1.5, 1.5);
            door.rotateY(-Math.PI/2);
            door.position.set(pos.x, pos.y, pos.z);
            bombSceneObj.door = door;
            if (transparent) {
                door.traverse((element) => {
                    if (element.material) {
                        element.material.transparent = true;
                        element.material.opacity = 0;
                    }
                });
            }

            if (bombExists) {
                const bomb = await loadModel(bombUrl, "bomb" + id, scene);
                bomb.scale.set(10, 10, 10);
                bomb.position.set(pos.x, pos.y, pos.z);
                bomb.translateY(-5.5);
                bomb.translateX(5);
                bombSceneObj.bomb = bomb;
                if (transparent) {
                    bomb.traverse((element) => {
                        if (element.material) {
                            element.material.transparent = true;
                            element.material.opacity = 0;
                        }
                    });
                }
            }

            bombSceneObj.droneObj = await addDrone(id, pos, bottom, transparent);

            return bombSceneObj;
        }

        function addArrow(pos, angle, length) {
            const arrowObj = {pos, angle, length};

            const lineGeometry = new THREE.CylinderGeometry(0.1, 0.1, length, 3, 1);
            const lineMaterial = new THREE.MeshStandardMaterial({color: 0x0000ff});
            const line = new THREE.Mesh(lineGeometry, lineMaterial);
            line.position.set(pos.x, pos.y, pos.z);
            line.rotateZ(angle);
            scene.add(line);
            arrowObj.line = line;

            const arrowGeometry = new THREE.ConeGeometry(0.4, 1, 10, 1);
            const arrowMaterial = new THREE.MeshStandardMaterial({color: 0x0000ff});
            const arrow = new THREE.Mesh(arrowGeometry, arrowMaterial);
            arrow.position.set(pos.x, pos.y, pos.z);
            arrow.rotateZ(angle);
            arrow.translateY(length/2);
            scene.add(arrow);
            arrowObj.arrow = arrow;

            return arrowObj;
        }

        async function updateMainSceneWithoutBomb(animationState) {
            animationState.bombSceneObj1.droneObj.dronePointLightHelper = changeLightHelperMaterial(animationState.bombSceneObj1.droneObj.dronePointLightHelper, animationState.quantumState1[0]);
            
            if (animationState.bombSceneObj1.droneObj2 == undefined){
                animationState.bombSceneObj1.droneObj2 = await addDrone("1", {x: 15, y: 10, z: 0}, true, false);
            }
            animationState.bombSceneObj1.droneObj2.dronePointLightHelper = changeLightHelperMaterial(animationState.bombSceneObj1.droneObj2.dronePointLightHelper, animationState.quantumState1[1]);

            changeOpacity({drone: animationState.bombSceneObj1.droneObj.drone}, animationState.quantumState1[0]);
            changeOpacity({drone: animationState.bombSceneObj1.droneObj2.drone}, animationState.quantumState1[1]);

            animationState.bombSceneObj1.droneObj.droneSpotLight.intensity = 100000*animationState.quantumState1[0];
            animationState.bombSceneObj1.droneObj2.droneSpotLight.intensity = 100000*animationState.quantumState1[1];

            if (animationState.quantumState1[0] <= 0 || animationState.quantumState1[1] >= 1) {
                removeGroups([animationState.bombSceneObj1.droneObj]);
                animationState.bombSceneObj1.botttom = true;
            }
            
            return animationState;
        }

        async function animateRotation(animationState) {
            if (!animationState.needToAnimateRotation) {
                return animationState;
            }
            if (animationState.bombSceneObj0.bottom && animationState.bombSceneObj1.bottom) {
                animationState.needToAnimateRotation = false;
                return animationState;
            }
            if (animationState.curStep == -1) {
                if (!animationState.bombSceneObj0.bottom) {
                    addBombScene("0", {x: -15, y: -15, z: 0}, false, true, true).then((value) => animationState.bombSceneObj00 = value);
                    addBombScene("0", {x: -35, y: -15, z: 0}, true, true, true).then((value) => animationState.bombSceneObj01 = value);
                }
                if (!animationState.bombSceneObj1.bottom) {
                    addBombScene("1", {x: 15, y: -15, z: 0}, false, false, true).then((value) => animationState.bombSceneObj10 = value);
                    addBombScene("1", {x: 35, y: -15, z: 0}, true, false, true).then((value) => animationState.bombSceneObj11 = value);
                }

                animationState.curStep += 1;
            }
            else if ((animationState.bombSceneObj00 && animationState.bombSceneObj01 || animationState.bombSceneObj0.bottom) && (animationState.bombSceneObj10 && animationState.bombSceneObj11 || animationState.bombSceneObj1.bottom)) {
                if (animationState.curStep <= 100) {
                    if (animationState.arrow00 && animationState.arrow01) {
                        removeGroups([animationState.arrow00, animationState.arrow01]);
                    }
                    
                    if (animationState.arrow10 && animationState.arrow11) {
                        removeGroups([animationState.arrow10,animationState.arrow11]);
                    }
                    
                    if (!animationState.bombSceneObj0.bottom) {
                        animationState.arrow00 = addArrow({x: -27.5+Math.sqrt(2)/2*5*(100-animationState.curStep)/100, y: -2.5+Math.sqrt(2)/2*5*(100-animationState.curStep)/100, z: 0}, 3*Math.PI/4, 10*animationState.curStep/100);
                        animationState.arrow01 = addArrow({x: -15, y: -2.5+3.75*(100-animationState.curStep)/100, z: 0}, Math.PI, 7.5*animationState.curStep/100);
                        changeOpacity(animationState.bombSceneObj00, animationState.curStep/100);
                        changeOpacity(animationState.bombSceneObj01, animationState.curStep/100);

                        if (animationState.curStep == 100) {
                            const dronePointLightHelper00 = new THREE.SpotLightHelper(animationState.bombSceneObj00.droneObj.droneSpotLight, 0x000000);
                            scene.add(dronePointLightHelper00);
                            animationState.bombSceneObj00.droneObj.dronePointLightHelper = dronePointLightHelper00;
                            
                            const dronePointLightHelper01 = new THREE.SpotLightHelper(animationState.bombSceneObj01.droneObj.droneSpotLight, 0x000000);
                            scene.add(dronePointLightHelper01);
                            animationState.bombSceneObj01.droneObj.dronePointLightHelper = dronePointLightHelper01;
                        }
                    }
                    if (!animationState.bombSceneObj1.bottom) {
                        animationState.arrow11 = addArrow({x: 27.5-Math.sqrt(2)/2*5*(100-animationState.curStep)/100, y: -2.5+Math.sqrt(2)/2*5*(100-animationState.curStep)/100, z: 0}, -3*Math.PI/4, 10*animationState.curStep/100);
                        animationState.arrow10 = addArrow({x: 15, y: -2.5+3.75*(100-animationState.curStep)/100, z: 0}, Math.PI, 7.5*animationState.curStep/100);
                        changeOpacity(animationState.bombSceneObj10, animationState.curStep/100);
                        changeOpacity(animationState.bombSceneObj11, animationState.curStep/100);

                        if (animationState.curStep == 100) {
                            const dronePointLightHelper10 = new THREE.SpotLightHelper(animationState.bombSceneObj10.droneObj.droneSpotLight, 0x000000);
                            scene.add(dronePointLightHelper10);
                            animationState.bombSceneObj10.droneObj.dronePointLightHelper = dronePointLightHelper10;

                            const dronePointLightHelper11 = new THREE.SpotLightHelper(animationState.bombSceneObj11.droneObj.droneSpotLight, 0x000000);
                            scene.add(dronePointLightHelper11);
                            animationState.bombSceneObj11.droneObj.dronePointLightHelper = dronePointLightHelper11;
                        }
                    }

                    animationState.curStep += 1;
                }
                if (animationState.curStep <= 200 && animationState.curStep > 100) {
                    if (animationState.curStep == 101) {
                        animationState.needToAnimateExplosion = [];
                        if (!animationState.bombSceneObj0.bottom) {
                            animationState.needToAnimateExplosion.push("01");
                        }
                    }

                    if (!animationState.bombSceneObj0.bottom && animationState.curStep <= 180) {
                        if (animationState.curStep%10 == 1) {
                            let result0;
                            
                            if (animationState.curframe0) {
                                scene.remove(animationState.curframe0);
                            }
                            result0 = randomSquareFrame(false, animationState.quantumState0[0]**2);
                            animationState.curframe0 = result0[0];
                            animationState.result0 = result0[1];
                        }
                    }

                    animationState.curStep += 1;
                }
                else if (animationState.curStep <= 300 && animationState.curStep > 200) {
                        if (!animationState.bombSceneObj0.bottom) {
                            if (animationState.curStep == 201) {
                                scene.remove(animationState.curframe0);
                                scene.remove(animationState.arrow00.line);
                                scene.remove(animationState.arrow00.arrow);
                                scene.remove(animationState.arrow01.line);
                                scene.remove(animationState.arrow01.arrow);
                                removeGroups([animationState.bombSceneObj0]);

                                if (animationState.result0) {
                                    removeGroups([animationState.bombSceneObj00]);
                                } else {
                                    removeGroups([animationState.bombSceneObj01]);
                                    scene.remove(animationState.explosion0);
                                    animationState.explosion0 = undefined;
                                }
                            }

                            if (animationState.result0) {
                                translateGroups([animationState.bombSceneObj01, {explosion0: animationState.explosion0}], {x: 20/100, y: 25/100, z: 0});
                            } else {
                                translateGroups([animationState.bombSceneObj00], {x: 0, y: 25/100, z:0});
                            }
                        }

                        if (!animationState.bombSceneObj1.bottom) {
                            if (animationState.curStep == 201) {
                                scene.remove(animationState.arrow10.line);
                                scene.remove(animationState.arrow10.arrow);
                                scene.remove(animationState.arrow11.line);
                                scene.remove(animationState.arrow11.arrow);
                                changeVisibility(animationState.bombSceneObj1, false);
                            }

                            translateGroups([animationState.bombSceneObj11], {x: -20/100, y: 25/100, z: 0});
                            translateGroups([animationState.bombSceneObj10], {x: 0, y: 25/100, z:0});
                        }

                    animationState.curStep += 1;
                }
                else if (animationState.curStep == 301){
                    if (animationState.newAnimStateResult == undefined) {
                        updateMainSceneWithoutBomb(animationState).then((value) => animationState.newAnimStateResult = value);
                        animationState.newAnimStateResult = "waiting"
                    }
                    if (animationState.newAnimStateResult != "waiting") {

                        if (!animationState.bombSceneObj0.bottom) {
                            if (animationState.result0) {
                                animationState.bombSceneObj0 = animationState.bombSceneObj01;
                                removeGroups([animationState.bombSceneObj00]);
                            } else {
                                animationState.bombSceneObj0 = animationState.bombSceneObj00;
                                removeGroups([animationState.bombSceneObj10]);
                            }
                        }

                        if (animationState.bombSceneObj0.bottom) {
                            importantValuesObj.quantumState0 = [0, 1];
                        }
                        else {
                            importantValuesObj.quantumState0 = [1, 0];
                        }
                        animationState.quantumState0 = importantValuesObj.quantumState0;
                        updateStateNumbers(animationState.quantumState0, animationState.quantumState1);

                        removeGroups([animationState.bombSceneObj10, animationState.bombSceneObj11]);
                        
                        changeVisibility(animationState.bombSceneObj1, true);
                        animationState = await animationState.newAnimStateResult;
                        animationState.curStep = -1;
                        animationState.needToAnimateRotation = false;

                        animationState.arrow00 = undefined;
                        animationState.arrow01 = undefined;
                        animationState.arrow10 = undefined;
                        animationState.arrow11 = undefined;

                        animationState.bombSceneObj00 = undefined;
                        animationState.bombSceneObj01 = undefined;
                        animationState.bombSceneObj10 = undefined;
                        animationState.bombSceneObj11 = undefined;
                        
                        animationState.newAnimStateResult = undefined;
                    }
                }
            }

            return animationState;
        }

        async function animateExplosion(animationState) {
            if (animationState.needToAnimateExplosion.length > 0) {
                if (animationState.curExplStep == -1 || animationState.curExplStep == undefined) {
                    animationState.curExplStep = 0;

                    if (animationState.needToAnimateExplosion.includes("0") || animationState.needToAnimateExplosion.includes("01")) {
                        loadModel(explosionUrl, "explosion", scene).then((value) => animationState.explosion0 = value);
                    }
                }
                else if (animationState.curExplStep == 0 && (animationState.explosion0 || !(animationState.needToAnimateExplosion.includes("0") || animationState.needToAnimateExplosion.includes("01")))) {
                    if (animationState.needToAnimateExplosion.includes("0") || animationState.needToAnimateExplosion.includes("01")) {
                        animationState.explosion0.scale.set(0.01, 0.01, 0.01);
                        animationState.explosion0.children[0].children[0].children[0].material.color={r: 1, g: 0.3, b:0};
                        scene.add(animationState.explosion0);
                        if (animationState.needToAnimateExplosion.includes("0")) {
                            animationState.explosion0.position.set(-15, 12, 0);
                        }
                        if (animationState.needToAnimateExplosion.includes("01")) {
                            animationState.explosion0.position.set(-35, -13, 0);
                        }
                    }

                    animationState.curExplStep += 1;
                }
                else if (animationState.curExplStep < 50 && (animationState.explosion0 || !(animationState.needToAnimateExplosion.includes("0") || animationState.needToAnimateExplosion.includes("01")))) {
                    let scale;
                    if (animationState.needToAnimateExplosion.includes("0") || animationState.needToAnimateExplosion.includes("01")) {
                        scale = animationState.explosion0.scale;
                        scale.x += (0.2-scale.x*0.5);
                        scale.y += (0.2-scale.y*0.5);
                        scale.z += (0.2-scale.z*0.5);
                    }                        
                    animationState.curExplStep += 1;
                }
                else if (animationState.curExplStep == 50) {
                    animationState.needToAnimateExplosion = [];
                    animationState.curExplStep = -1;
                }
            }
            return animationState;
        }

        initScene();
        let bombSceneObj0 = await addBombScene("0", {x: -15, y: 10, z: 0}, false, true);
        let bombSceneObj1 = await addBombScene("1", {x: 15, y: 10, z: 0}, false, false);
        let outsideDoor0 = await loadModel(outsideDoorUrl, "outside_door_0", scene);
        outsideDoor0.scale.set(6, 6, 6);
        outsideDoor0.rotateY(-Math.PI/2);
        outsideDoor0.translateZ(23);
        outsideDoor0.translateY(9);
        let outsideDoor1 = await loadModel(outsideDoorUrl, "outside_door_1", scene);
        outsideDoor1.scale.set(6, 6, 6);
        outsideDoor1.rotateY(-Math.PI/2);
        outsideDoor1.translateZ(-7);
        outsideDoor1.translateY(9);
        let animationState = {bombSceneObj0, bombSceneObj1, curStep: -1, needToAnimateRotation: false, quantumState0: importantValuesObj.quantumState0,
            quantumState1: importantValuesObj.quantumState1, needToAnimateExplosion: [], curExplStep: -1, outsideDoor0, outsideDoor1};
  
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

        async function animate() {
            if (importantValuesObj.needsReset == true) {
                importantValuesObj.measure = false;
                importantValuesObj.quantumState0 = [1, 0];
                importantValuesObj.quantumState1 = [1, 0];
                importantValuesObj.needToRotate = false;
                importantValuesObj.needToRotateWithAnim = false;
                importantValuesObj.needsReset = false;
                scene.clear();
                initScene();
                bombSceneObj0 = await addBombScene("0", {x: -15, y: 10, z: 0}, false, true);
                bombSceneObj1 = await addBombScene("1", {x: 15, y: 10, z: 0}, false, false);
                outsideDoor0 = await loadModel(outsideDoorUrl, "outside_door_0", scene);
                outsideDoor0.scale.set(6, 6, 6);
                outsideDoor0.rotateY(-Math.PI/2);
                outsideDoor0.translateZ(23);
                outsideDoor0.translateY(9);
                outsideDoor1 = await loadModel(outsideDoorUrl, "outside_door_1", scene);
                outsideDoor1.scale.set(6, 6, 6);
                outsideDoor1.rotateY(-Math.PI/2);
                outsideDoor1.translateZ(-7);
                outsideDoor1.translateY(9);
                animationState = {bombSceneObj0, bombSceneObj1, curStep: -1, needToAnimateRotation: false, quantumState0: importantValuesObj.quantumState0,
                    quantumState1: importantValuesObj.quantumState1, needToAnimateExplosion: [], curExplStep: -1, outsideDoor0, outsideDoor1};
            }


            if (importantValuesObj.measure) {
                if (!((animationState.quantumState1[0] == 0 || animationState.quantumState1[1] == 0) && 
                (animationState.quantumState0[0] == 0 || animationState.quantumState0[1] == 0))) {
                    importantValuesObj.measure = false;
                    return;
                }

                if (animationState.curMeasStep == undefined){
                    animationState.curMeasStep = -1;
                }
                if (animationState.curMeasStep < 50) {
                    if (!(animationState.bombSceneObj0.bottom)) {
                        animationState.outsideDoor0.getObjectByName("leftDoor").translateX(-0.75/50);
                        animationState.outsideDoor0.getObjectByName("rightDoor").translateX(0.75/50);
                    }
                    animationState.outsideDoor1.getObjectByName("leftDoor").translateX(-0.75/50);
                    animationState.outsideDoor1.getObjectByName("rightDoor").translateX(0.75/50);
                    animationState.curMeasStep += 1;
                }
                if (animationState.curMeasStep == 50) {
                    animationState.curMeasStep = -1;
                    importantValuesObj.measure = false;
                }
            }

            if (animationState.bombSceneObj0.bottom && !animationState.doorExpl0) {
                if (animationState.curMeasStep == undefined || animationState.curMeasStep == -1){
                    animationState.curMeasStep = -1;
                    if (animationState.bombSceneObj0.bottom && !animationState.doorExpl0) {
                        if (animationState.explosion0) {
                            scene.remove(animationState.explosion0);
                            animationState.explosion0 = undefined;
                        }
                        animationState.needToAnimateExplosion.push("0");
                    }
                }
                if (animationState.curMeasStep < 25) {
                    if (animationState.bombSceneObj0.bottom && !animationState.doorExpl0) {
                        animationState.outsideDoor0.getObjectByName("leftDoor").rotateY(-Math.PI/6/25);
                        animationState.outsideDoor0.getObjectByName("leftDoor").rotateZ(-Math.PI/10/25);
                        animationState.outsideDoor0.getObjectByName("leftDoor").translateX(-0.3/25);
                        animationState.outsideDoor0.getObjectByName("leftDoor").translateZ(0.6/25);
                        animationState.outsideDoor0.getObjectByName("leftDoor").translateY(-0.3/25);
                        animationState.outsideDoor0.getObjectByName("rightDoor").rotateY(Math.PI/6/25);
                        animationState.outsideDoor0.getObjectByName("rightDoor").rotateZ(Math.PI/6/25);
                        animationState.outsideDoor0.getObjectByName("rightDoor").translateX(0.3/25);
                        animationState.outsideDoor0.getObjectByName("rightDoor").translateZ(0.5/25);
                        animationState.outsideDoor0.getObjectByName("rightDoor").translateY(-0.3/25);
                    }

                    animationState.curMeasStep += 1;
                }
                if (animationState.curMeasStep == 25) {
                    animationState.curMeasStep = -1;
                    if (animationState.bombSceneObj0.bottom && !animationState.doorExpl0) {
                        animationState.doorExpl0 = true;
                    }
                }
            }

            if (importantValuesObj.needToRotate) {
                if (animationState.curStep == -1) {
                    if (!animationState.bombSceneObj0.bottom) {
                        importantValuesObj.quantumState0 = rotateState(importantValuesObj.quantumState0, importantValuesObj.rotationAngle);
                        animationState.quantumState0 = importantValuesObj.quantumState0;
                    }
                    if (animationState.quantumState1[0] <= 1 && animationState.quantumState1[1] >= 0) {
                        importantValuesObj.quantumState1 = rotateState(importantValuesObj.quantumState1, importantValuesObj.rotationAngle);
                        animationState.quantumState1 = importantValuesObj.quantumState1;
                    }
                    updateStateNumbers(animationState.quantumState0, animationState.quantumState1);
                    updateMainSceneWithoutBomb(animationState).then((value) => animationState.newAnimStateResult = value);
                    animationState.newAnimStateResult = "waiting";
                }
                if (animationState.curStep < 5) {
                    animationState.curStep += 1;
                }
                else if (animationState.newAnimStateResult != "waiting") {
                    animationState = await animationState.newAnimStateResult;
                    animationState.newAnimStateResult = undefined;
                    if (Math.random() > animationState.quantumState0[0]**2 && !animationState.bombSceneObj0.bottom) {
                        animationState.bombSceneObj0 = rotateDrone(animationState.bombSceneObj0, true);
                    }
                    if (animationState.bombSceneObj0.bottom) {
                        importantValuesObj.quantumState0 = [0, 1];
                    }
                    else {
                        importantValuesObj.quantumState0 = [1, 0];
                    }
                    animationState.quantumState0 = importantValuesObj.quantumState0;
                    updateStateNumbers(animationState.quantumState0, animationState.quantumState1);
                    importantValuesObj.needToRotate = false;
                    animationState.curStep = -1;
                }
            }

            if (importantValuesObj.needToRotateWithAnim) {
                if (!animationState.bombSceneObj0.bottom) {
                    importantValuesObj.quantumState0 = rotateState(importantValuesObj.quantumState0, importantValuesObj.rotationAngle);
                    animationState.quantumState0 = importantValuesObj.quantumState0;
                }
                if (!animationState.bombSceneObj1.bottom) {
                    importantValuesObj.quantumState1 = rotateState(importantValuesObj.quantumState1, importantValuesObj.rotationAngle);
                    animationState.quantumState1 = importantValuesObj.quantumState1;
                }
                updateStateNumbers(animationState.quantumState0, animationState.quantumState1);
                animationState.needToAnimateRotation = true;
                
                importantValuesObj.needToRotateWithAnim = false;
            }

            animationState = await animateRotation(animationState);
            animationState = await animateExplosion(animationState);

            renderer.clear();
            renderer.render(backgroundScene, camera);
            renderer.render(scene, camera);
        }

        renderer.setAnimationLoop(animate);
    });
</script>

<div id="bomb-detection-container">
    <div class="controls-container">
        <label for="rotation-angle">Rotation Angle: </label>
        <input type="number" id="rotation-angle" on:change={(event) => rotationAngleUpdate(event.target.valueAsNumber)} value=0.1/>
        <img  id="rotation-gate-image" src={pdfUrlRotationGate} alt="Rotation Gate"/>
        <button id="rotate-button" on:click={rotateClicked}>Rotate</button>
        <button id="rotate-with-animation-button" on:click={rotateWithAnimationClicked}>Rotate With Animation</button>
        <button id="measure-button" on:click={measureClicked}>Measure</button>
        <div>*You can only measure</div>
        <div>when states are final.</div>
        <button id="reset-button" on:click={resetClicked}>Reset</button>
    </div>
</div>

<style>
    .controls-container {
        position: absolute;
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    #rotation-gate-image {
        border-radius: 1000px;
        width: 10vw;
    }
</style>