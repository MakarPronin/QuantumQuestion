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

    const catUrl = new URL("../assets/models/cat.glb", import.meta.url);
    const deadCatUrl = new URL("../assets/models/dead_cat.glb", import.meta.url);
    const hammerUrl = new URL("../assets/models/hammer.glb", import.meta.url);
    const potionUrl = new URL("../assets/models/potion.glb", import.meta.url);
    const potionBrokenUrl = new URL("../assets/models/potion_broken.glb", import.meta.url);

    let state = {oldClassicalBoxWithCat: undefined, scene: undefined, classicalState: Math.random() > 0.5, particlesArr: new Array(10).fill(undefined), particlesDirectionArr: new Array(10).fill(undefined)};

    async function reset() {
        state = state = {oldClassicalBoxWithCat: undefined, scene: state.scene, classicalState: Math.random() > 0.5, particlesArr: new Array(10).fill(undefined), particlesDirectionArr: new Array(10).fill(undefined)};

        state.scene.clear();

        initScene();
        state.oldClassicalBoxWithCat = await addBoxWithCat(-1, false, 2);
    }

    function getRandomNumberInRange(rangesArr) {
        const randomIndex = Math.floor(Math.random() * rangesArr.length);
        const a = rangesArr[randomIndex][0];
        const b = rangesArr[randomIndex][1];
        let n = Math.random()*(b-a);
        n += a;
        return n;
    }

    function animateDecoherenceParticles(step=0.05, ballSize=0.3, numParticles = 10, center={x: 0, y: 0, z: 0}, range=18) {
        if (state.particlesArr.reduce((prevValue, curValue) => {
                    if (curValue != undefined) {
                        prevValue++;
                    }
                    return prevValue;
                }, 0) < numParticles) {
            const x = getRandomNumberInRange([[center.x-range+ballSize/2, center.x+range-ballSize/2]]);
            const y = getRandomNumberInRange([[center.y-range+ballSize/2, center.y+range-ballSize/2]]);
            let z;
            if ((x >= center.x-12-ballSize/2 && x <= center.x+12+ballSize/2) &&
            (y >= center.y-10-ballSize/2 && y <= center.y+10+ballSize/2)){
                z = getRandomNumberInRange([[center.z-range+ballSize/2, center.z-10-ballSize/2], [center.z+10+ballSize/2, center.z+range-ballSize/2]]);
            }
            else {
                z = getRandomNumberInRange([[center.z-range+ballSize/2, center.z+range-ballSize/2]]);
            }
            addDecoherenceParticle(x, y, z, ballSize);
        }
        for (let i in state.particlesArr.reduce((prevValue, curValue, curIndex) => {
                if (curValue != undefined) {
                    prevValue.push(curIndex);
                }
                return prevValue;
            }, [])) {
            if ((state.particlesArr[i].sphere.position.x >= center.x-9-ballSize/2 && state.particlesArr[i].sphere.position.x <= center.x+9+ballSize/2) &&
                (state.particlesArr[i].sphere.position.y >= center.y-7-ballSize/2 && state.particlesArr[i].sphere.position.y <= center.y+7+ballSize/2) &&
                (state.particlesArr[i].sphere.position.z >= center.z-7-ballSize/2 && state.particlesArr[i].sphere.position.z <= center.z+7+ballSize/2)
                ) {
                state.particlesDirectionArr[i] = true;
                if (state.oldClassicalBoxWithCat != undefined) {
                    removeObjectFromScene(state.oldClassicalBoxWithCat);
                    addBoxWithCat(-1, false, (state.classicalState) ? 1 : 0);
                    state.oldClassicalBoxWithCat = undefined;
                }
                addLabelToParticle(i);
            }
            const xDiff = center.x - state.particlesArr[i].sphere.position.x;
            const yDiff = center.y - state.particlesArr[i].sphere.position.y;
            const zDiff = center.z - state.particlesArr[i].sphere.position.z;
            const norm = Math.sqrt(xDiff**2 + yDiff**2 + zDiff**2);
            if (state.particlesDirectionArr[i]) {
                moveGroups([state.particlesArr[i]], -step*xDiff/norm, -step*yDiff/norm, -step*zDiff/norm);
            }
            else {
                moveGroups([state.particlesArr[i]], step*xDiff/norm, step*yDiff/norm, step*zDiff/norm);
            }
            if ((state.particlesArr[i].sphere.position.x < center.x-range+ballSize/2 || state.particlesArr[i].sphere.position.x > center.x+range-ballSize/2) ||
                (state.particlesArr[i].sphere.position.y < center.y-range+ballSize/2 || state.particlesArr[i].sphere.position.y > center.y+range-ballSize/2) ||
                (state.particlesArr[i].sphere.position.z < center.z-range+ballSize/2 || state.particlesArr[i].sphere.position.z > center.z+range-ballSize/2)
                ) {
                state.particlesDirectionArr[i] = undefined;
                removeObjectFromScene(state.particlesArr[i]);
                state.particlesArr[i] = undefined;
            }
        }
    }

    function addDecoherenceParticle(x, y, z, ballSize) {
        const sphereGeometry = new THREE.SphereGeometry(ballSize, 10, 10);
        const sphereMaterial = new THREE.MeshStandardMaterial({color: 0xaaaaaa});
        const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial);
        sphere.position.set(x, y, z);
        state.scene.add(sphere);
        state.particlesArr[state.particlesArr.findIndex(el => el == undefined)] = {sphere: sphere};
        state.particlesDirectionArr[state.particlesDirectionArr.findIndex(el => el == undefined)] = false;
    }

    function addLabelToParticle(i) {
        if (state.particlesArr[i]["label"] != undefined) {
            return;
        }
        let text;
        if (state.classicalState) {
            text = "Dead";
        }
        else {
            text = "Alive";
        }
        const label = new TextSprite({
            text: text,
            fontSize: 2,
            color: '#000000',
        });

        label.position.set(state.particlesArr[i].sphere.position.x, state.particlesArr[i].sphere.position.y, state.particlesArr[i].sphere.position.z);
        state.scene.add(label);
        state.particlesArr[i]["label"] = label;
    }

    function prepareStateValToOutput(expr) {
            expr = math.format(math.evaluate(expr), 3).toString();

            expr = expr.replaceAll(" ", "");

            expr = expr.slice(0, 5);
            
            while (expr.length < 5) {
                expr = expr + " ";
            }

            return expr;
    }

    function moveGroups(objArr, x, y, z) {
        for (let obj of objArr) {
            for (let key of Object.keys(obj)) {
                if (obj[key] && (obj[key].type == "Mesh" || obj[key].type == "Sprite")) {
                    obj[key].applyMatrix4(new THREE.Matrix4().makeTranslation(x, y, z));
                }
            }
        }
    }

    function removeObjectFromScene(obj) {
        if (obj == undefined || typeof obj !== 'object') {
            return;
        }
        if (obj.isObject3D) {
            state.scene.remove(obj);
            return;
        }
        for (let key of Object.keys(obj)) {
            removeObjectFromScene(obj[key]);
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

    function addStateToBlochSphere(alphaExpr, betaExpr, color, scale=0.6) {
        function addArrow(horizAngle=-Math.PI/2, vertAngle, radius=10, color=0xff0000, fromCenter=0, labelText, sideLabel = false) {
            const arrowObj = {horizAngle, vertAngle, radius, color, fromCenter, labelText};

            const lineGeometry = new THREE.CylinderGeometry(0.3*scale, 0.3*scale, (radius*2-1)*scale, 3, 1);
            const lineMaterial = new THREE.MeshStandardMaterial({color: color});
            const line = new THREE.Mesh(lineGeometry, lineMaterial);
            line.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
            line.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
            line.translateY((-0.5 + fromCenter)*scale);
            state.scene.add(line);
            arrowObj.line = line;

            const arrowGeometry = new THREE.ConeGeometry(1*scale, 1*scale, 10, 1);
            const arrowMaterial = new THREE.MeshStandardMaterial({color: color});
            const arrow = new THREE.Mesh(arrowGeometry, arrowMaterial);
            arrow.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
            arrow.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
            arrow.translateY((radius-0.5 + fromCenter)*scale);
            state.scene.add(arrow);
            arrowObj.arrow = arrow;

            let colorString = color.toString(16);
            while (colorString.length < 6) {
                colorString = "0" + colorString;
            }
            colorString = "#" + colorString;

            let label = new TextSprite({
                text: labelText,
                fontSize: 1*scale,
                color: colorString,
                fontFamily: "monospace",
            });
            label.material.depthTest = false;
            label.material.needsUpdate = true;
            label.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
            label.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
            if (sideLabel) {
                label.translateY(fromCenter*scale);
                label.translateX(2*scale);
            }
            else {
                label.translateY(1.4*(radius + fromCenter)*scale);
            }
            state.scene.add(label);
            arrowObj.label = label;

            return arrowObj;
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

        let arrowObj;

        let alpha = math.evaluate(alphaExpr);
        let beta = math.evaluate(betaExpr);

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

        const toOutput = [prepareStateValToOutput(alpha.toString()), prepareStateValToOutput(beta.toString())];

        arrowObj = addArrow((horizAngle - Math.PI/2 + 4*Math.PI)%(2*Math.PI), vertAngle, 10, color, 0, "⎧"+toOutput[0]+"⎫\n⎩"+toOutput[1]+"⎭");

        return arrowObj;
    }

    function addBlochSphere(scale=0.6) {
        const blochSphereObj = {};

        const circleGeometryXY = new THREE.TorusGeometry(10*scale, 0.1*scale, 3, 100, 2*Math.PI);
        const circleMaterialXY = new THREE.MeshStandardMaterial({color: 0x000000});
        const circleXY = new THREE.Mesh(circleGeometryXY, circleMaterialXY);
        state.scene.add(circleXY);
        blochSphereObj.circleXY = circleXY;

        const circleGeometryXZ = new THREE.TorusGeometry(10*scale, 0.1*scale, 3, 100, 2*Math.PI);
        const circleMaterialXZ = new THREE.MeshStandardMaterial({color: 0x000000});
        const circleXZ = new THREE.Mesh(circleGeometryXZ, circleMaterialXZ);
        circleXZ.rotateY(Math.PI/2);
        state.scene.add(circleXZ);
        blochSphereObj.circleXZ = circleXZ;

        const circleGeometryYZ = new THREE.TorusGeometry(10*scale, 0.1*scale, 3, 100, 2*Math.PI);
        const circleMaterialYZ = new THREE.MeshStandardMaterial({color: 0x000000});
        const circleYZ = new THREE.Mesh(circleGeometryYZ, circleMaterialYZ);
        circleYZ.rotateX(Math.PI/2);
        state.scene.add(circleYZ);
        blochSphereObj.circleYZ = circleYZ;

        const lineGeometryX = new THREE.CylinderGeometry(0.1*scale, 0.1*scale, 24*scale, 3, 1);
        const lineMaterialX = new THREE.MeshStandardMaterial({color: 0x000000});
        const lineX = new THREE.Mesh(lineGeometryX, lineMaterialX);
        lineX.rotateZ(Math.PI/2);
        state.scene.add(lineX);
        blochSphereObj.lineX = lineX;

        const arrowGeometryX0 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
        const arrowMaterialX0 = new THREE.MeshStandardMaterial({color: 0x000000});
        const arrowX0 = new THREE.Mesh(arrowGeometryX0, arrowMaterialX0);
        arrowX0.translateX(12*scale);
        arrowX0.rotateZ(-Math.PI/2);
        state.scene.add(arrowX0);
        blochSphereObj.arrowX0 = arrowX0;

        const arrowGeometryX1 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
        const arrowMaterialX1 = new THREE.MeshStandardMaterial({color: 0x000000});
        const arrowX1 = new THREE.Mesh(arrowGeometryX1, arrowMaterialX1);
        arrowX1.translateX(-12*scale);
        arrowX1.rotateZ(Math.PI/2);
        state.scene.add(arrowX1);
        blochSphereObj.arrowX1 = arrowX1;

        const lineGeometryY = new THREE.CylinderGeometry(0.1*scale, 0.1*scale, 24*scale, 3, 1);
        const lineMaterialY = new THREE.MeshStandardMaterial({color: 0x000000});
        const lineY = new THREE.Mesh(lineGeometryY, lineMaterialY);
        state.scene.add(lineY);
        blochSphereObj.lineY = lineY;

        const arrowGeometryY0 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
        const arrowMaterialY0 = new THREE.MeshStandardMaterial({color: 0x000000});
        const arrowY0 = new THREE.Mesh(arrowGeometryY0, arrowMaterialY0);
        arrowY0.translateY(12*scale);
        state.scene.add(arrowY0);
        blochSphereObj.arrowY0 = arrowY0;

        const arrowGeometryY1 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
        const arrowMaterialY1 = new THREE.MeshStandardMaterial({color: 0x000000});
        const arrowY1 = new THREE.Mesh(arrowGeometryY1, arrowMaterialY1);
        arrowY1.translateY(-12*scale);
        arrowY1.rotateX(Math.PI);
        state.scene.add(arrowY1);
        blochSphereObj.arrowY1 = arrowY1;

        const lineGeometryZ = new THREE.CylinderGeometry(0.1*scale, 0.1*scale, 24*scale, 3, 1);
        const lineMaterialZ = new THREE.MeshStandardMaterial({color: 0x000000});
        const lineZ = new THREE.Mesh(lineGeometryZ, lineMaterialZ);
        lineZ.rotateX(Math.PI/2);
        state.scene.add(lineZ);
        blochSphereObj.lineZ = lineZ;

        const arrowGeometryZ0 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
        const arrowMaterialZ0 = new THREE.MeshStandardMaterial({color: 0x000000});
        const arrowZ0 = new THREE.Mesh(arrowGeometryZ0, arrowMaterialZ0);
        arrowZ0.translateZ(12*scale);
        arrowZ0.rotateX(Math.PI/2);
        state.scene.add(arrowZ0);
        blochSphereObj.arrowZ0 = arrowZ0;

        const arrowGeometryZ1 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
        const arrowMaterialZ1 = new THREE.MeshStandardMaterial({color: 0x000000});
        const arrowZ1 = new THREE.Mesh(arrowGeometryZ1, arrowMaterialZ1);
        arrowZ1.translateZ(-12*scale);
        arrowZ1.rotateX(-Math.PI/2);
        state.scene.add(arrowZ1);
        blochSphereObj.arrowZ1 = arrowZ1;

        let labelX0 = new TextSprite({
            text: '|𝑖⟩',
            fontSize: 2*scale,
            color: '#000000',
        });
        labelX0.material.depthTest = false;
        labelX0.material.needsUpdate = true;
        labelX0.translateX(14*scale);
        state.scene.add(labelX0);
        blochSphereObj.labelX0 = labelX0;
        
        let labelX1 = new TextSprite({
            text: '|-𝑖⟩',
            fontSize: 2*scale,
            color: '#000000',
        });
        labelX1.material.depthTest = false;
        labelX1.material.needsUpdate = true;
        labelX1.translateX(-14*scale);
        state.scene.add(labelX1);
        blochSphereObj.labelX1 = labelX1;

        let labelY0 = new TextSprite({
            text: '|0⟩',
            fontSize: 2*scale,
            color: '#000000',
        });
        labelY0.material.depthTest = false;
        labelY0.material.needsUpdate = true;
        labelY0.translateY(14*scale);
        state.scene.add(labelY0);
        blochSphereObj.labelY0 = labelY0;

        let labelY1 = new TextSprite({
            text: '|1⟩',
            fontSize: 2*scale,
            color: '#000000',
        });
        labelY1.material.depthTest = false;
        labelY1.material.needsUpdate = true;
        labelY1.translateY(-14*scale);
        state.scene.add(labelY1);
        blochSphereObj.labelY1 = labelY1;

        let labelZ0 = new TextSprite({
            text: '|+⟩',
            fontSize: 2*scale,
            color: '#000000',
        });
        labelZ0.material.depthTest = false;
        labelZ0.material.needsUpdate = true;
        labelZ0.translateZ(14*scale);
        state.scene.add(labelZ0);
        blochSphereObj.labelZ0 = labelZ0;
        
        let labelZ1 = new TextSprite({
            text: '|-⟩',
            fontSize: 2*scale,
            color: '#000000',
        });
        labelZ1.material.depthTest = false;
        labelZ1.material.needsUpdate = true;
        labelZ1.translateZ(-14*scale);
        state.scene.add(labelZ1);
        blochSphereObj.labelZ1 = labelZ1;

        return blochSphereObj;
    }

    function initScene() {
        const scene = state.scene;
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

        let labelDecoherence = new TextSprite({
            text: 'Decoherence',
            fontSize: 2,
            color: '#000000',
        });
        labelDecoherence.translateY(25);
        scene.add(labelDecoherence);
    }

    async function addBoxWithCat(pos=1, open=false, dead=0) {
        const boxWithCatObj = {};
        let x = 0;
        if (pos == 0) {
            x = -40;
        }
        if (pos == 1) {
            x = -18;
        }
        if (pos == 2) {
            x = 18;
        }
        if (pos == -1) {
            x = 0;
        }
        const box = addBox(x, open, dead);
        if (Array.isArray(box)) {
            box.forEach((el, index) => boxWithCatObj[`box-plane-${index}`] = el);
        }
        else {
            boxWithCatObj.box = box;
        }
        if (dead == 0 || dead == 2) {
            const cat = await loadModel(catUrl, "cat"+pos.toString(), state.scene);
            cat.translateX(x-6);
            cat.translateY(-7);
            cat.scale.set(4, 4, 4);
            const potion = await loadModel(potionUrl, "potion"+pos.toString(), state.scene);
            potion.scale.set(0.007, 0.007, 0.007);
            potion.translateX(x);
            potion.translateY(0);
            const liquidGreenGlassMaterial = potion.children[0].children[0].children[0].children[0].children[0].material;
            liquidGreenGlassMaterial.depthTest = false;
            liquidGreenGlassMaterial.needsUpdate = true;
            const hammer = await loadModel(hammerUrl, "hammer"+pos.toString(), state.scene);
            hammer.translateX(x+4);
            hammer.translateY(3);
            hammer.rotateX(-Math.PI/2);
            hammer.rotateY(-2/4*Math.PI);
            hammer.scale.set(0.3, 0.3, 0.3);
            boxWithCatObj.cat0 = cat;
            boxWithCatObj.potion0 = potion;
            boxWithCatObj.hammer0 = hammer;
        }
        if (dead == 1 || dead == 2) {
            const cat = await loadModel(deadCatUrl, "deadCat"+pos.toString(), state.scene);
            cat.translateX(x-13);
            cat.translateY(0);
            cat.rotateY(-8*Math.PI/10);
            cat.scale.set(8, 8, 8);
            const potion = await loadModel(potionBrokenUrl, "potionBroken"+pos.toString(), state.scene);
            potion.scale.set(0.007, 0.007, 0.007);
            potion.translateX(x);
            potion.translateY(0);
            const liquidGreenGlassMaterial = potion.children[0].children[0].children[0].children[0].children[0].material;
            liquidGreenGlassMaterial.depthTest = false;
            liquidGreenGlassMaterial.needsUpdate = true;
            const hammer = await loadModel(hammerUrl, "hammer"+pos.toString(), state.scene);
            hammer.translateX(x+4);
            hammer.translateY(3);
            hammer.rotateX(-Math.PI/2);
            hammer.rotateY(-3.5/4*Math.PI);
            hammer.scale.set(0.3, 0.3, 0.3);
            boxWithCatObj.cat1 = cat;
            boxWithCatObj.potion1 = potion;
            boxWithCatObj.hammer1 = hammer;
        }
        const blochSphere = addBlochSphere(0.2);
        boxWithCatObj.blochSphere = blochSphere;
        let stateBloch;
        if (dead == 0) {
            stateBloch = addStateToBlochSphere("1", "0", 0xff0000, 0.2);
        }
        if (dead == 1) {
            stateBloch = addStateToBlochSphere("0", "1", 0xff0000, 0.2);
        }
        if (dead == 2) {
            stateBloch = addStateToBlochSphere("1/sqrt(2)", "1/sqrt(2)", 0xff0000, 0.2);
        }
        boxWithCatObj.stateBloch = stateBloch;
        if (pos == 0 || pos == 1) {
            const space = await loadModel(spaceUrl, "space"+pos.toString(), state.scene);
            boxWithCatObj.space = space;
            space.translateY(10);
            space.translateX(x-5);
            space.translateZ(4);
            space.scale.set(4, 4, 4);

            if (demoState < 2) {
                let labelSpace = new TextSprite({
                    text: '|Universe⟩ - entangled qubits in superposition forming the universe',
                    fontSize: 1,
                    color: '#000000',
                });
                boxWithCatObj.labelSpace = labelSpace;
                labelSpace.translateX(x);
                labelSpace.translateY(18);
                state.scene.add(labelSpace);

                let labelBox = new TextSprite({
                    text: '|cat⟩ - entangled qubits in superposition forming the cat',
                    fontSize: 1,
                    color: '#000000',
                });
                boxWithCatObj.labelBox = labelBox;
                labelBox.translateX(x);
                labelBox.translateY(9);
                state.scene.add(labelBox);
            }
        }
        rotateGroups([blochSphere, stateBloch], [0, 1, 0], Math.PI/2);
        moveGroups([blochSphere, stateBloch], x+7, 4, 0);
        return boxWithCatObj;
    }

    function addBox(x=0, open=false, dead=0) {
        const width = 18;
        const height = 14;
        const depth = 14;
        let opacity;
        let color;
        if (dead == 0) {
            color = 0x0000aa;
            opacity = 0.2;
        }
        if (dead == 1) {
            color = 0x00aa00;
            opacity = 0.4;
        }
        if (dead == 2) {
            color = 0x00ffaa;
            opacity = 0.3;
        }
        if (!open) {
            const boxGeometry = new THREE.BoxGeometry(18, height, depth, 1, 1, 1);
            const boxMaterial = new THREE.MeshStandardMaterial({color, opacity, transparent: true, side: THREE.DoubleSide});
            const box = new THREE.Mesh(boxGeometry, boxMaterial);
            box.translateX(x);
            state.scene.add(box);
            return box;
        }
        else {
            // Material
            const material = new THREE.MeshStandardMaterial({ color, opacity, transparent: true, side: THREE.DoubleSide });

            // Box construction using individual planes
            const createPlane = (w, h, posX, posY, posZ, rotX, rotY, rotZ) => {
                const geometry = new THREE.PlaneGeometry(w, h);
                const plane = new THREE.Mesh(geometry, material);
                plane.position.set(posX, posY, posZ);
                plane.translateX(x);
                plane.rotation.set(rotX, rotY, rotZ);
                state.scene.add(plane);
                return plane;
            };

            // Creating planes for the box
            const topOpen = createPlane(width/2, depth, -width/2 + 1/4*width*math.cos(Math.PI/6), height/2 + 1/4*width*math.sin(Math.PI/6), 0, Math.PI / 2, Math.PI / 6, 0);
            const topClosed = createPlane(width/2, depth, width/4, height/2, 0, Math.PI / 2, 0, 0);
            const bottom = createPlane(width, depth, 0, -height / 2, 0, Math.PI / 2, 0, 0);
            const left = createPlane(depth, height, -width / 2, 0, 0, 0, Math.PI / 2, 0);
            const right = createPlane(depth, height, width / 2, 0, 0, 0, Math.PI / 2, 0);
            const back = createPlane(width, height, 0, 0, -depth / 2, 0, 0, 0);
            const front = createPlane(width, height, 0, 0, depth / 2, 0, 0, 0);
            return [topOpen, topClosed, bottom, left, right, back, front];
        }
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
        state.scene = scene;
        const mousePosition = new THREE.Vector2();
        renderer.setSize(width, height);
        renderer.shadowMap.enabled = true;
        document.getElementById("decoherence-container").appendChild(renderer.domElement);

        let camera = new THREE.PerspectiveCamera(75, width / height, 0.01, 10000000);
        const orbit = new OrbitControls(camera, renderer.domElement);
        camera.position.set(0, 0, 40);
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

        function renderSceneElements() {
            animateDecoherenceParticles(0.05, 0.3, 10, {x: 0, y: 0, z: 0}, 30);
        }

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


        initScene();
        state.oldClassicalBoxWithCat = await addBoxWithCat(-1, false, 2);

        function animate() {
            renderSceneElements();
            renderer.clear();
            renderer.render(backgroundScene, camera);
            renderer.render(scene, camera);
        }

        renderer.setAnimationLoop(animate);
    });
</script>

<div id="decoherence-container">
    <div class="controls-container">
        <button id="reset-button" style="background-color:rgba(255, 160, 160);" on:click={reset}>Reset</button>
    </div>
</div>

<style>
    .controls-container {
        position: absolute;
        display: flex;
        flex-direction: column;
        align-items: center;
    }
</style>