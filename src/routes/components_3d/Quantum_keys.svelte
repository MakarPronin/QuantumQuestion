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

    let state = {qubitsArr: [], qubitSceneObjArr: [], moving: false, movingTextY: false, movingTextY1: false, textSceneObjArr: [], textPositionArr: [], qubitPositionArr: [], scene: undefined, firstTextReachedEnd: false};

    let inputObj = {x: "", y: "", x1: "", y1: "", x2: "", y2: "", eve: false};

    let demoState = 0;

    let demoTextArr = ["Initialize x and y", "Initialize y'", "Save changes to x, y, y'", "Transmit qubits", "Transmit y", "Transmit y'", "Finished"];

    function reset() {
        state = {qubitsArr: [], qubitSceneObjArr: [], moving: false, movingTextY: false, movingTextY1: false, textSceneObjArr: [], textPositionArr: [], qubitPositionArr: [], scene: state.scene, firstTextReachedEnd: false};

        inputObj = {x: "", y: "", x1: "", y1: "", x2: "", y2: "", eve: false, transMatrix: {"0-0": "1", "0-1": "0", "1-0": "0", "1-1": "1"}};

        demoState = 0;

        state.scene.clear();

        initScene();
    }

    function nextDemoStep() {
        if (demoState == 0) {
            inputObj.x = "";
            inputObj.y = "";
            for (let i = 0; i < 10; i++) {
                inputObj.x += Math.random() < 0.5 ? "0" : "1";
                inputObj.y += Math.random() < 0.5 ? "0" : "1";
            }
        }
        if (demoState == 1) {
            inputObj.y1 = "";
            for (let i = 0; i < 10; i++) {
                inputObj.y1 += Math.random() < 0.5 ? "0" : "1";
            }
        }
        if (demoState == 2) {
            const inputKeys = ["x", "y", "y1"];
            for (let key of inputKeys) {
                let newStr = "";
                for (let char of inputObj[key]) {
                    if (char != '0' && char != '1') {
                        newStr += Math.random() < 0.5 ? "0" : "1";
                    }
                    else {
                        newStr += char;
                    }
                }
                if (newStr.length < 10) {
                    newStr += Math.random() < 0.5 ? "0" : "1";
                }
                inputObj[key] = newStr;
            }
        }
        if (demoState == 3) {
            for (let i = 0; i < 10; i++) {
                if (inputObj.y.charAt(i) == '0') {
                    if (inputObj.x.charAt(i) == '0') {
                        state.qubitsArr.push(['1', '0']);
                    }
                    else {
                        state.qubitsArr.push(['0', '1']);
                    }
                }
                else {
                    if (inputObj.x.charAt(i) == '0') {
                        state.qubitsArr.push(['1/sqrt(2)', '1/sqrt(2)']);
                    }
                    else {
                        state.qubitsArr.push(['1/sqrt(2)', '-1/sqrt(2)']);
                    }
                }
            }
            state.moving = true;
        }
        if (demoState == 4) {
            state.qubitsArr = [];
            state.qubitSceneObjArr = [];
            state.qubitPositionArr = [];
            state.movingTextY = true;
        }
        if (demoState == 5) {
            state.textPositionArr = [];
            state.textSceneObjArr = [];
            state.firstTextReachedEnd = false;
            state.movingTextY1 = true;
        }
        if (demoState == 6) {
            state.textPositionArr = [];
            state.textSceneObjArr = [];
            state.firstTextReachedEnd = false;
        }
        demoState++;
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

    function applyTransform(alphaExpr, betaExpr, matrix=[['1', '0'], ['0', '1']]) {
        const newAlphaNum = math.evaluate("(("+matrix[0][0]+')*('+alphaExpr+"))+(("+matrix[0][1]+')*('+betaExpr+"))");
        const newBetaNum = math.evaluate("(("+matrix[1][0]+')*('+alphaExpr+"))+(("+matrix[1][1]+')*('+betaExpr+"))");

        return [newAlphaNum.toString(), newBetaNum.toString()];
    }

    function addMeasurement(person="Bob", axis='Z') {
        let i;
        if (person=='Bob') {
            i = state.qubitPositionArr.findIndex((el) => el > 100);
        }
        if (person == "Eve") {
            i = state.qubitPositionArr.findIndex((el) => el == 50);
            inputObj.y2 += (axis == "Z" ? "0" : "1");
        }
        const alphaExpr = state.qubitsArr[i][0];
        const betaExpr = state.qubitsArr[i][1];

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

        const newAlpha = math.evaluate(`(${result[0]})*conj(${result[0]})`);

        if (person == 'Bob') {
            if (newAlpha >= Math.random()) {
                inputObj.x1 += "0";
            }
            else {
                inputObj.x1 += "1";
            }
            state.qubitPositionArr[i] = undefined;
        }
        if (person == 'Eve') {
            if (newAlpha >= Math.random()) {
                inputObj.x2 += "0";
            }
            else {
                inputObj.x2 += "1";
            }
        }
        const obj = state.qubitSceneObjArr[i];
        removeGroups([obj.blochSphere, obj.stateArrow])
        state.qubitSceneObjArr[i] = undefined;
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

    function removeGroups(objArr) {
        for (let obj of objArr) {
            for (let key of Object.keys(obj)) {
                if (obj[key] && (obj[key].type == "Mesh" || obj[key].type == "Sprite")) {
                    state.scene.remove(obj[key]);
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

    function addNewQubit(alphaExpr, betaExpr, pos="Alice") {
        if (pos == "Alice") {
            const obj = {blochSphere: addBlochSphere(0.2), stateArrow: addStateToBlochSphere(alphaExpr, betaExpr, 0xff0000, 0.2)};
            state.qubitSceneObjArr.push(obj);
            state.qubitPositionArr.push(0);
            rotateGroups([obj.blochSphere, obj.stateArrow], [0, 1, 0], Math.PI/2)
            moveGroups([obj.blochSphere, obj.stateArrow], -35, 9, 0);
        }
        if (pos == "Eve") {
            const obj = {blochSphere: addBlochSphere(0.2), stateArrow: addStateToBlochSphere(alphaExpr, betaExpr, 0xff0000, 0.2)};
            const i = state.qubitPositionArr.findIndex((el) => el == 50);
            state.qubitSceneObjArr[i] = obj;
            state.qubitsArr[i] = [alphaExpr, betaExpr];
            rotateGroups([obj.blochSphere, obj.stateArrow], [0, 1, 0], Math.PI/2)
            moveGroups([obj.blochSphere, obj.stateArrow], 0, -2.55, 0);
            state.qubitPositionArr[i] = 50;
        }
    }

    function addNewText(text) {
        const textSceneObj = new TextSprite({
            text: text,
            fontSize: 2,
            color: '#000000',
        });
        textSceneObj.material.depthTest = false;
        textSceneObj.material.needsUpdate = true;
        state.scene.add(textSceneObj);
        state.textSceneObjArr.push(textSceneObj);
        if (state.movingTextY) {
            state.textPositionArr.push(0);
            moveGroups([{obj: textSceneObj}], -35, 20, 0);
        }
        if (state.movingTextY1) {
            state.textPositionArr.push(100);
            moveGroups([{obj: textSceneObj}], 35, 20, 0);
        }
    }

    function sendQubit(stateArg) {
        let alphaExpr;
        let betaExpr;

        if (stateArg == "0") {
            alphaExpr = "1";
            betaExpr = "0";
        }
        if (stateArg == "1") {
            alphaExpr = "0";
            betaExpr = "1";
        }
        if (stateArg == "+") {
            alphaExpr = "1/sqrt(2)";
            betaExpr = "1/sqrt(2)";
        }
        if (stateArg == "-") {
            alphaExpr = "1/sqrt(2)";
            betaExpr = "-1/sqrt(2)";
        }

        addNewQubit(alphaExpr, betaExpr, "Eve");
        state.moving = true;
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

        const labelAlice = new TextSprite({
            text: 'Alice',
            fontSize: 2,
            color: '#000000',
        });
        labelAlice.translateX(-43);
        labelAlice.translateY(25);
        scene.add(labelAlice);

        const labelBob = new TextSprite({
            text: 'Bob',
            fontSize: 2,
            color: '#000000',
        });
        labelBob.translateX(43);
        labelBob.translateY(25);
        scene.add(labelBob);

        const labelEve = new TextSprite({
            text: 'Eve',
            fontSize: 2,
            color: '#000000',
        });
        labelEve.translateY(-8);
        scene.add(labelEve);

        const labelClassicalChannel = new TextSprite({
            text: 'Classical Authenticated Channel',
            fontSize: 2,
            color: '#000000',
        });
        labelClassicalChannel.translateY(25);
        scene.add(labelClassicalChannel);

        const labelQuantumChannel = new TextSprite({
            text: 'Quantum Channel',
            fontSize: 2,
            color: '#000000',
        });
        labelQuantumChannel.translateY(14);
        scene.add(labelQuantumChannel);

        const classicalChannelGeometry = new THREE.CylinderGeometry(3, 3, 70, 20, 1, true);
        const classicalChannelMaterial = new THREE.MeshStandardMaterial({transparent: true, opacity: 0.3, color: 0x000000, side: THREE.DoubleSide});
        const classicalChannel = new THREE.Mesh(classicalChannelGeometry, classicalChannelMaterial);
        classicalChannel.rotateZ(Math.PI/2);
        classicalChannel.translateX(20);
        scene.add(classicalChannel);

        const eveChannelGeometry0 = new THREE.CylinderGeometry(1, 1, 6, 20, 1, true);
        const eveChannelMaterial0 = new THREE.MeshStandardMaterial({transparent: true, opacity: 0.3, color: 0x000000, side: THREE.DoubleSide});
        const eveChannel0 = new THREE.Mesh(eveChannelGeometry0, eveChannelMaterial0);
        eveChannel0.translateY(20);
        eveChannel0.translateZ(-5);
        eveChannel0.rotateX(-Math.PI/2);
        eveChannel0.rotateY(Math.PI/2);
        scene.add(eveChannel0);

        const eveChannelGeometry1 = new THREE.CylinderGeometry(1, 1, 20, 20, 1, true);
        const eveChannelMaterial1 = new THREE.MeshStandardMaterial({transparent: true, opacity: 0.3, color: 0x000000, side: THREE.DoubleSide});
        const eveChannel1 = new THREE.Mesh(eveChannelGeometry1, eveChannelMaterial1);
        eveChannel1.translateY(11);
        eveChannel1.translateZ(-7);
        scene.add(eveChannel1);

        const quantumChannelGeometry = new THREE.CylinderGeometry(3, 3, 70, 20, 1, true);
        const quantumChannelMaterial = new THREE.MeshStandardMaterial({transparent: true, opacity: 0.3, color: 0x0000a0, side: THREE.DoubleSide});
        const quantumChannel = new THREE.Mesh(quantumChannelGeometry, quantumChannelMaterial);
        quantumChannel.rotateZ(Math.PI/2);
        quantumChannel.translateX(9);
        scene.add(quantumChannel);

        const quantumChannelGeometry0 = new THREE.CylinderGeometry(3, 3, 10, 20, 1, true);
        const quantumChannelMaterial0 = new THREE.MeshStandardMaterial({transparent: true, opacity: 0.3, color: 0x0000a0, side: THREE.DoubleSide});
        const quantumChannel0 = new THREE.Mesh(quantumChannelGeometry0, quantumChannelMaterial0);
        quantumChannel0.translateX(-8);
        quantumChannel0.translateY(5);
        quantumChannel0.rotateZ(Math.PI/4);
        scene.add(quantumChannel0);

        const quantumChannelGeometry1 = new THREE.CylinderGeometry(3, 3, 10, 20, 1, true);
        const quantumChannelMaterial1 = new THREE.MeshStandardMaterial({transparent: true, opacity: 0.3, color: 0x0000a0, side: THREE.DoubleSide});
        const quantumChannel1 = new THREE.Mesh(quantumChannelGeometry1, quantumChannelMaterial1);
        quantumChannel1.translateX(8);
        quantumChannel1.translateY(5);
        quantumChannel1.rotateZ(-Math.PI/4);
        scene.add(quantumChannel1);
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
        document.getElementById("quantum-keys-container").appendChild(renderer.domElement);

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

        function moveQubits(step=1) {
            if (inputObj.eve) {
                for (let i = 0; i < 10; i++) {
                    if (state.qubitSceneObjArr[i] == undefined) {
                        continue;
                    }
                    const obj = state.qubitSceneObjArr[i];
                    state.qubitPositionArr[i] += step/2;
                    moveGroups([obj.blochSphere, obj.stateArrow], 0.7*step/2, 0, 0);
                    if (state.qubitPositionArr[i] > 33 && state.qubitPositionArr[i] < 50) {
                        moveGroups([obj.blochSphere, obj.stateArrow], 0, -0.7*step/2, 0);
                    }
                    if (state.qubitPositionArr[i] == 50) {
                        state.moving = false;
                    }
                    if (state.qubitPositionArr[i] > 50 && state.qubitPositionArr[i] < 67) {
                        moveGroups([obj.blochSphere, obj.stateArrow], 0, 0.7*step/2, 0);
                    }
                }
            }
            else {
                for (let i = 0; i < 10; i++) {
                    if (state.qubitSceneObjArr[i] == undefined) {
                        continue;
                    }
                    const obj = state.qubitSceneObjArr[i];
                    state.qubitPositionArr[i] += step/2;
                    moveGroups([obj.blochSphere, obj.stateArrow], 0.7*step/2, 0, 0);
                }
            }
        }

        function moveText(step=1) {
            let direction = 1;
            if  (state.movingTextY1) {
                direction = -1;
            }
            for (let i = 0; i < 10; i++) {
                if (state.textSceneObjArr[i] == undefined) {
                    continue;
                }
                const obj = state.textSceneObjArr[i];
                state.textPositionArr[i] += direction*step/2;
                moveGroups([{obj}], direction*0.7*step/2, 0, 0);
            }
        }

        function addY() {
            let i;
            let charsY = [...inputObj.y];
            let charsY1 = [...inputObj.y1];
            let charsY2 = [...inputObj.y2];
            let charsX = [...inputObj.x];
            let charsX1 = [...inputObj.x1];
            let charsX2 = [...inputObj.x2];
            if (state.movingTextY) {
                i = state.textPositionArr.findIndex((el) => el > 100);
                if (i != -1) {
                    if (charsY[i] == charsY1[i]) {
                        charsY1[i] = charsY1[i] == "1" ? "𝟏" : "𝟎";
                        if (charsX[i] != charsX1[i]) {
                            charsX1[i] = "X";
                        }
                        else {
                            charsX1[i] = charsX1[i] == "1" ? "𝟏" : "𝟎";
                        }
                    }
                    inputObj.y1 = charsY1.join('');
                    inputObj.x1 = charsX1.join('');
                }
            }
            if (state.movingTextY1) {
                i = state.textPositionArr.findIndex((el) => el < 0);
                if (i != -1) {
                    if (charsY[i] == convertIrregularCharacters(charsY1[i])) {
                        charsY[i] = charsY[i] == "1" ? "𝟏" : "𝟎";
                        charsX[i] = charsX[i] == "1" ? "𝟏" : "𝟎";
                    }
                    inputObj.y = charsY.join('');
                    inputObj.x = charsX.join('');
                }
            }
            if (state.movingTextY && inputObj.eve && i == -1) {
                i = state.textPositionArr.findIndex((el) => el == 50);
                if (i != -1) {
                    if (charsY[i] == charsY2[i]) {
                        charsY2[i] = charsY2[i] == "1" ? "𝟏" : "𝟎";
                        charsX2[i] = charsX2[i] == "1" ? "𝟏" : "𝟎";
                    }
                    inputObj.y2 = charsY2.join('');
                    inputObj.x2 = charsX2.join('');
                }
            }
            if (state.textPositionArr[i] != 50 && i != -1) {
                const obj = state.textSceneObjArr[i];
                removeGroups([{obj}]);
                state.textSceneObjArr[i] = undefined;
                state.textPositionArr[i] = undefined;
                if (i == 0) {
                    state.firstTextReachedEnd = true;
                }
            }
        }

        function convertIrregularCharacters(char) {
            if (char == "𝟏" || char == "1") {
                return "1";
            }
            else {
                return "0";
            }
        }

        function stopMovementOnFinish() {
            if (state.moving) {
                if (state.qubitSceneObjArr.length == 10 && state.qubitSceneObjArr[9] == undefined) {
                    state.moving = false;
                }
            }
            if (state.movingTextY) {
                if (state.textSceneObjArr.length == 10 && state.textSceneObjArr[9] == undefined) {
                    state.movingTextY = false;
                }
            }
            if (state.movingTextY1) {
                if (state.textSceneObjArr.length == 10 && state.textSceneObjArr[9] == undefined) {
                    state.movingTextY1 = false;
                }
            }
        }

        function renderSceneElements() {
            if (state.moving) {
                if (state.qubitSceneObjArr[0] == undefined && inputObj.x1 == "") {
                    addNewQubit(state.qubitsArr[0][0], state.qubitsArr[0][1]);
                }
                for (let i = 0; i < 10; i++) {
                    if (i != 0 && state.qubitSceneObjArr[i - 1] != undefined && state.qubitSceneObjArr[i] == undefined && state.qubitPositionArr[i - 1] > 10) {
                        addNewQubit(state.qubitsArr[i][0], state.qubitsArr[i][1]);
                    }
                    if (state.qubitPositionArr[i] > 100) {
                        addMeasurement("Bob", inputObj.y1.charAt(i) == "0" ? "Z" : "X");
                    }
                }
                moveQubits();
            }
            if (state.movingTextY) {
                if (state.textSceneObjArr[0] == undefined && state.firstTextReachedEnd == false) {
                    addNewText(convertIrregularCharacters([...inputObj.y][0]));
                }
                for (let i = 0; i < 10; i++) {
                    if (i != 0 && state.textSceneObjArr[i - 1] != undefined && state.textSceneObjArr[i] == undefined && state.textPositionArr[i - 1] > 10) {
                        addNewText(convertIrregularCharacters([...inputObj.y][i]));
                    }
                    addY();
                }
                moveText();
            }
            if (state.movingTextY1) {
                if (state.textSceneObjArr[0] == undefined && state.firstTextReachedEnd == false) {
                    addNewText(convertIrregularCharacters([...inputObj.y1][0]));
                }
                for (let i = 0; i < 10; i++) {
                    if (i != 0 && state.textSceneObjArr[i - 1] != undefined && state.textSceneObjArr[i] == undefined && state.textPositionArr[i - 1] < 90) {
                        addNewText(convertIrregularCharacters([...inputObj.y1][i]));
                    }
                    addY();
                }
                moveText();
            }
            stopMovementOnFinish();
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

        function animate() {
            renderSceneElements();
            renderer.clear();
            renderer.render(backgroundScene, camera);
            renderer.render(scene, camera);
        }

        renderer.setAnimationLoop(animate);
    });
</script>

<div id="quantum-keys-container">
    <div class="controls-container" id="controls-container-left">
        <span>Alice:</span>
        <div>
            <label for="x-input">x = </label>
            <input class="text-input" id="x-input" bind:value={inputObj.x} maxlength="10" disabled={demoState != 2}/>
        </div>
        <div>
            <label for="y-input">y = </label>
            <input class="text-input" id="y-input" bind:value={inputObj.y} maxlength="10" disabled={demoState != 2}/>
        </div>

        <span style="margin-top: 2rem;">Eve:</span>
        <div>
            <label for="x2-input">x = </label>
            <input class="text-input" id="x2-input" bind:value={inputObj.x2} maxlength="10" disabled/>
        </div>
        <div>
            <label for="y2-input">y = </label>
            <input class="text-input" id="y2-input" bind:value={inputObj.y2} maxlength="10" disabled/>
        </div>
        <div>
            <label for="listen-checkbox">Listen</label>
            <input type="checkbox" id="listen-checkbox" class="checkbox-round" bind:value={inputObj.eve} disabled={demoState > 0}/>
        </div>
        <button id="measure-Z" on:click={() => addMeasurement("Eve", 'Z')} disabled={demoState != 4 || state.moving || state.qubitSceneObjArr[state.qubitPositionArr.findIndex(el => el == 50)] == undefined}>Measure in Z basis</button>
        <button id="measure-X" on:click={() => addMeasurement("Eve", 'X')}  disabled={demoState != 4 || state.moving || state.qubitSceneObjArr[state.qubitPositionArr.findIndex(el => el == 50)] == undefined}>Measure in X basis</button>
        <button id="send-0" on:click={() => sendQubit('0')} disabled={demoState != 4 || state.moving || state.qubitSceneObjArr[state.qubitPositionArr.findIndex(el => el == 50)] != undefined || (state.qubitSceneObjArr.length == 10 && state.qubitSceneObjArr[9] == undefined && state.qubitSceneObjArr[8] == undefined)}>Send qubit |0⟩</button>
        <button id="send-1" on:click={() => sendQubit('1')} disabled={demoState != 4 || state.moving || state.qubitSceneObjArr[state.qubitPositionArr.findIndex(el => el == 50)] != undefined || (state.qubitSceneObjArr.length == 10 && state.qubitSceneObjArr[9] == undefined && state.qubitSceneObjArr[8] == undefined)}>Send qubit |1⟩</button>
        <button id="send-plus" on:click={() => sendQubit('+')} disabled={demoState != 4 || state.moving || state.qubitSceneObjArr[state.qubitPositionArr.findIndex(el => el == 50)] != undefined || (state.qubitSceneObjArr.length == 10 && state.qubitSceneObjArr[9] == undefined && state.qubitSceneObjArr[8] == undefined)}>Send qubit |+⟩</button>
        <button id="send-minus" on:click={() => sendQubit('-')} disabled={demoState != 4 || state.moving || state.qubitSceneObjArr[state.qubitPositionArr.findIndex(el => el == 50)] != undefined || (state.qubitSceneObjArr.length == 10 && state.qubitSceneObjArr[9] == undefined && state.qubitSceneObjArr[8] == undefined)}>Send qubit |-⟩</button>
        <span style="margin-top: 2rem;">Demo:</span>
        <button id="demo-button" style="background-color:rgba(160, 255, 160);" on:click={nextDemoStep} disabled={demoState > 5 || state.moving || state.movingTextY || state.movingTextY1 || state.qubitPositionArr.findIndex(el => el == 50) != -1}>{demoTextArr[demoState]}</button>
        <button id="reset-button" style="background-color:rgba(255, 160, 160);" on:click={reset}>Reset</button>
    </div>
    <div class="controls-container" id="controls-container-right">
        <span>Bob:</span>
        <div>
            <label for="x1-input">x' = </label>
            <input class="text-input" id="x1-input" bind:value={inputObj.x1} disabled/>
        </div>
        <div>
            <label for="y1-input">y' = </label>
            <input class="text-input" id="y1-input" bind:value={inputObj.y1} maxlength="10" disabled={demoState != 2}/>
        </div>
    </div>
</div>

<style>
    .controls-container {
        position: absolute;
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    #controls-container-right {
        right: 0%;
    }
</style>