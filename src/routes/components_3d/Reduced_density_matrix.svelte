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

    let stateArr = [{state: [(1/Math.sqrt(2)).toString(), '0', '0', (1/Math.sqrt(2)).toString()], leftMeas: false, rightMeas: false}];

    let reducedDensityA = {"0-0": "0.5", "0-1": "0", "1-0": "0", "1-1": "0.5"};
    let reducedDensityB = {"0-0": "0.5", "0-1": "0", "1-0": "0", "1-1": "0.5"};

    let state = {};

    let inputObj = {matrixAlice: {}, matrixBob: {}, forgetAlice: false, forgetBob: false};

    function applyTransformTwoQubits(matrix=[['1', '0', '0', '0'], ['0', '1', '0', '0'], ['0', '0', '1', '0'], ['0', '0', '0', '1']]) {
        if (state.leftMeas || state.rightMeas) {
            return;
        }
        const s = state.state;
        let newState = [];
        for (let i = 0; i < 4; i++) {
            newState.push((math.evaluate(
               "(("+matrix[i][0]+')*('+s[0]+"))+\
                (("+matrix[i][1]+')*('+s[1]+"))+\
                (("+matrix[i][2]+')*('+s[2]+"))+\
                (("+matrix[i][3]+')*('+s[3]+"))"
            )).toString());
        }
        stateArr.push({state: newState, leftMeas: state.leftMeas, rightMeas: state.rightMeas, leftMeasResult: state.leftMeasResult, rightMeasResult: state.rightMeasResult});
    }

    function applyTransformToSide(matrix=[['1', '0'], ['0', '1']], side=false) {
        const s = state.state;
        if ((state.leftMeas && side) || (state.rightMeas && !side)) {
            const newAlphaNum = math.evaluate("(("+matrix[0][0]+')*('+s[0]+"))+(("+matrix[0][1]+')*('+s[1]+"))");
            const newBetaNum = math.evaluate("(("+matrix[1][0]+')*('+s[0]+"))+(("+matrix[1][1]+')*('+s[1]+"))");

            const newState = [newAlphaNum.toString(), newBetaNum.toString()];
            stateArr.push({state: newState, leftMeas: state.leftMeas, rightMeas: state.rightMeas, leftMeasResult: state.leftMeasResult, rightMeasResult: state.rightMeasResult});
        }

        let newMatrix;
        if (side) {
            newMatrix = [
                [matrix[0][0], '0', matrix[0][1], '0'],
                ['0', matrix[0][0], '0', matrix[0][1]],
                [matrix[1][0], '0', matrix[1][1], '0'],
                ['0', matrix[1][0], '0', matrix[1][1]]
            ];
        }
        else {
            newMatrix = [
                [matrix[0][0], matrix[0][1], '0', '0'],
                [matrix[1][0], matrix[1][1], '0', '0'],
                ['0', '0', matrix[0][0], matrix[0][1]],
                ['0', '0', matrix[1][0], matrix[1][1]],
            ];
        }
        applyTransformTwoQubits(newMatrix);
    }

    function reset() {
        stateArr = [{state: [(1/Math.sqrt(2)).toString(), '0', '0', (1/Math.sqrt(2)).toString()], leftMeas: false, rightMeas: false}];

        reducedDensityA = {"0-0": "0.5", "0-1": "0", "1-0": "0", "1-1": "0.5"};
        reducedDensityB = {"0-0": "0.5", "0-1": "0", "1-0": "0", "1-1": "0.5"};

        state = {};

        inputObj = {matrixAlice: {}, matrixBob: {}, forgetAlice: false, forgetBob: false};
    }

    function applyGateClicked(side) {
        function conjugateTranspose(matrix) {
            let result = [["0", "0"], ["0", "0"]];

            for (let i = 0; i < 2; i++) {
                for (let j = 0; j < 2; j++) {
                    let element = matrix[i][j];
                    let conjugate = math.evaluate(`conj(${element})`).toString();
                    result[j][i] = conjugate;
                }
            }

            return result;
        }

        function applyTransformToMatrix(transformMatrix, matrix) {
            let result = [["0", "0"], ["0", "0"]];

            const tm = transformMatrix;
            const m = matrix;

            result[0][0] = math.evaluate(`((${tm[0][0]})*(${m[0][0]}))+((${tm[0][1]})*(${m[1][0]}))`).toString();
            result[0][1] = math.evaluate(`((${tm[0][0]})*(${m[0][1]}))+((${tm[0][1]})*(${m[1][1]}))`).toString();
            result[1][0] = math.evaluate(`((${tm[1][0]})*(${m[0][0]}))+((${tm[1][1]})*(${m[1][0]}))`).toString();
            result[1][1] = math.evaluate(`((${tm[1][0]})*(${m[0][1]}))+((${tm[1][1]})*(${m[1][1]}))`).toString();

            return result;
        }

        let transformMatrix;
        if (!side) {
            transformMatrix = [
                [inputObj.matrixAlice["0-0"], inputObj.matrixAlice["0-1"]],
                [inputObj.matrixAlice["1-0"], inputObj.matrixAlice["1-1"]],
            ];
        }
        else {
            transformMatrix = [
                [inputObj.matrixBob["0-0"], inputObj.matrixBob["0-1"]],
                [inputObj.matrixBob["1-0"], inputObj.matrixBob["1-1"]],
            ];
        }
        const conjTransMatrix = conjugateTranspose(transformMatrix);

        applyTransformToSide(transformMatrix, side);

        let matrixToUpdate;

        if (!side) {
            matrixToUpdate = reducedDensityA;
        }
        else {
            matrixToUpdate = reducedDensityB;
        }
        let tempMatrix = [[matrixToUpdate["0-0"], matrixToUpdate["0-1"]], [matrixToUpdate["1-0"], matrixToUpdate["1-1"]]];
        tempMatrix = applyTransformToMatrix(applyTransformToMatrix(transformMatrix, tempMatrix), conjTransMatrix);
        matrixToUpdate["0-0"] = math.round(math.evaluate(tempMatrix[0][0].toString()), 3).toString();
        matrixToUpdate["0-1"] = math.round(math.evaluate(tempMatrix[0][1].toString()), 3).toString();
        matrixToUpdate["1-0"] = math.round(math.evaluate(tempMatrix[1][0].toString()), 3).toString();
        matrixToUpdate["1-1"] = math.round(math.evaluate(tempMatrix[1][1].toString()), 3).toString();
    }
    
    function reverseTensorProduct(stateArr) {
        if (state.leftMeas && state.rightMeas) {
            return {leftState: undefined, rightState: undefined};
        }
        if (state.leftMeas) {
            return {leftState: undefined, rightState: [stateArr[0], stateArr[1]]};
        }
        if (state.rightMeas) {
            return {leftState: [stateArr[0], stateArr[1]], rightState: undefined};
        }
        let a;
        let b;
        let c;
        let d;
        if (math.evaluate(stateArr[0]) != 0) {
            a = math.evaluate(`sqrt(1/(1+((${stateArr[2]})*conj(${stateArr[2]}))/((${stateArr[0]})*conj(${stateArr[0]}))))`);
            b = math.evaluate(`${stateArr[2]}/${stateArr[0]}*${a}`);
            c = math.evaluate(`${stateArr[0]}/${a}`);
            d = math.evaluate(`${stateArr[1]}/${a}`);
        }
        else {
            if (math.evaluate(stateArr[1]) == 0) {
                a = 0;
                b = 1;
                c = math.evaluate(stateArr[2]);
                d = math.evaluate(stateArr[3]);
            }
            else {
                a = math.evaluate(stateArr[1]);
                b = math.evaluate(stateArr[3]);
                c = 0;
                d = 1;
            }
        }
        if (math.abs(1 - (math.evaluate(`${a.toString()}*conj(${a.toString()})`) + math.evaluate(`${b.toString()}*conj(${b.toString()})`))) < 10**-6 &&
                math.abs(1 - (math.evaluate(`${c.toString()}*conj(${c.toString()})`) + math.evaluate(`${d.toString()}*conj(${d.toString()})`))) < 10**-6 &&
                math.evaluate(`abs(${a.toString()}*${c.toString()} - ${stateArr[0]})`) < 10**-6 &&
                math.evaluate(`abs(${a.toString()}*${d.toString()} - ${stateArr[1]})`) < 10**-6 &&
                math.evaluate(`abs(${b.toString()}*${c.toString()} - ${stateArr[2]})`) < 10**-6 &&
                math.evaluate(`abs(${b.toString()}*${d.toString()} - ${stateArr[3]})`) < 10**-6
            ){
            return {leftState: [a.toString(), b.toString()], rightState: [c.toString(), d.toString()]};
        }
        else {
            return {leftState: undefined, rightState: undefined};
        }
    }

    function reducedDensityMatrixForSingeQubit(state){
        const reducedDensityMatrix = {};
        reducedDensityMatrix['0-0'] = math.round(math.evaluate(`(${state[0]})*conj(${state[0]})`), 3).toString();
        reducedDensityMatrix['0-1'] = math.round(math.evaluate(`(${state[0]})*conj(${state[1]})`), 3).toString();
        reducedDensityMatrix['1-0'] = math.round(math.evaluate(`(${state[1]})*conj(${state[0]})`), 3).toString();
        reducedDensityMatrix['1-1'] = math.round(math.evaluate(`(${state[1]})*conj(${state[1]})`), 3).toString();
        return reducedDensityMatrix;
    }

    function getDensityMatrices(state) {
        function stateToDensityMatrix(state) {
            const densityMatrix = {};
            for (let i = 0; i < 4; i++) {
                for (let j = 0; j < 4; j++) {
                    densityMatrix[`${i}-${j}`] = math.evaluate(`(${state[i]})*conj(${state[j]})`).toString();
                }
            }
            return densityMatrix;
        }

        function reducedDensityMatrixA(densityMatrix) {
            const reducedDensityMatrix = {};
            reducedDensityMatrix['0-0'] = math.round(math.evaluate(`(${densityMatrix['0-0']}) + (${densityMatrix['1-1']})`), 3).toString();
            reducedDensityMatrix['0-1'] = math.round(math.evaluate(`(${densityMatrix['0-2']}) + (${densityMatrix['1-3']})`), 3).toString();
            reducedDensityMatrix['1-0'] = math.round(math.evaluate(`(${densityMatrix['2-0']}) + (${densityMatrix['3-1']})`), 3).toString();
            reducedDensityMatrix['1-1'] = math.round(math.evaluate(`(${densityMatrix['2-2']}) + (${densityMatrix['3-3']})`), 3).toString();
            return reducedDensityMatrix;
        }

        function reducedDensityMatrixB(densityMatrix) {
            const reducedDensityMatrix = {};
            reducedDensityMatrix['0-0'] = math.round(math.evaluate(`(${densityMatrix['0-0']}) + (${densityMatrix['2-2']})`), 3).toString();
            reducedDensityMatrix['0-1'] = math.round(math.evaluate(`(${densityMatrix['0-1']}) + (${densityMatrix['2-3']})`), 3).toString();
            reducedDensityMatrix['1-0'] = math.round(math.evaluate(`(${densityMatrix['1-0']}) + (${densityMatrix['3-2']})`), 3).toString();
            reducedDensityMatrix['1-1'] = math.round(math.evaluate(`(${densityMatrix['1-1']}) + (${densityMatrix['3-3']})`), 3).toString();
            return reducedDensityMatrix;
        }

        let reducedDensityA = undefined;
        let reducedDensityB = undefined;
        let densityMatrix;

        if (!state.leftMeas && !state.rightMeas) {
            densityMatrix = stateToDensityMatrix(state.state);
        }


        if (!state.leftMeas) {
            if (state.rightMeas) {
                reducedDensityA = reducedDensityMatrixForSingeQubit(state.state);
            }
            else {
                reducedDensityA = reducedDensityMatrixA(densityMatrix);
            }
        }

        if (!state.rightMeas) {
            if (state.leftMeas) {
                reducedDensityB = reducedDensityMatrixForSingeQubit(state.state);
            }
            else {
                reducedDensityB = reducedDensityMatrixB(densityMatrix);
            }
        }

        return {reducedDensityA, reducedDensityB};
    }

    function measureQubit(side=false, basis="Z") {
        function multByConj(expr) {
            return `(${expr}*conj(${expr}))`;
        }

        if (basis == 'Z') {
            applyTransformToSide(
                [["1", "0"],
                ["0", "1"]], side
            );
        }
        if (basis == 'X') {
            applyTransformToSide(
                [["1/sqrt(2)", "1/sqrt(2)"],
                ["1/sqrt(2)", "-1/sqrt(2)"]], side
            );
        }
        if (basis == 'Y') {
            applyTransformToSide(
                [["1/sqrt(2)", "-i/sqrt(2)"],
                 ["1/sqrt(2)", "i/sqrt(2)"]], side
            );
        }
        const newState = stateArr[stateArr.length - 1];

        if (!side) {
            newState.leftMeas = true;
            let alphaPrL;
            if (state.rightMeas) {
                alphaPrL = math.evaluate(multByConj(newState.state[0]))
            }
            else {
                alphaPrL = math.evaluate(`(${multByConj(newState.state[0])})+(${multByConj(newState.state[1])})`);
            }
            const n = Math.random();
            if (basis == 'Z') {
                if (n <= alphaPrL) {
                    newState.leftMeasResult = ['1', '0'];
                }
                else {
                    newState.leftMeasResult = ['0', '1'];
                }
            }
            if (basis == 'X') {
                if (n <= alphaPrL) {
                    newState.leftMeasResult = [(1/Math.sqrt(2)).toString(), (1/Math.sqrt(2)).toString()];
                }
                else {
                    newState.leftMeasResult = [(1/Math.sqrt(2)).toString(), (-1/Math.sqrt(2)).toString()];
                }
            }
            if (basis == 'Y') {
                if (n <= alphaPrL) {
                    newState.leftMeasResult = [(math.evaluate(`1/sqrt(2)`)).toString(), (math.evaluate(`i/sqrt(2)`)).toString()];
                }
                else {
                    newState.leftMeasResult = [(math.evaluate(`1/sqrt(2)`)).toString(), (math.evaluate(`-i/sqrt(2)`)).toString()];
                }
            }

            if (state.rightMeas) { 
                newState.state = [];
            }
            else {
                if (n <= alphaPrL) {
                    const normalization = '(' + (math.evaluate(`sqrt((${multByConj(newState.state[0])})+${multByConj(newState.state[1])})`)).toString() + ')';
                    newState.state = [math.evaluate(`(${newState.state[0]})/${normalization}`).toString(), math.evaluate(`(${newState.state[1]})/${normalization}`).toString()];
                }
                else {
                    const normalization = '(' + (math.evaluate(`sqrt((${multByConj(newState.state[2])})+${multByConj(newState.state[3])})`)).toString() + ')';
                    newState.state = [math.evaluate(`(${newState.state[2]})/${normalization}`).toString(), math.evaluate(`(${newState.state[3]})/${normalization}`).toString()];
                }
            }
        }
        else {
            newState.rightMeas = true;
            let alphaPrR;
            if (state.leftMeas) {
                alphaPrR = math.evaluate(multByConj(newState.state[0]))
            }
            else {
                alphaPrR = math.evaluate(`(${multByConj(newState.state[0])})+(${multByConj(newState.state[2])})`);
            }
            const n = Math.random();
            if (basis == 'Z') {
                if (n <= alphaPrR) {
                    newState.rightMeasResult = ['1', '0'];
                }
                else {
                    newState.rightMeasResult = ['0', '1'];
                }
            }
            if (basis == 'X') {
                if (n <= alphaPrR) {
                    newState.rightMeasResult = [(1/Math.sqrt(2)).toString(), (1/Math.sqrt(2)).toString()];
                }
                else {
                    newState.rightMeasResult = [(1/Math.sqrt(2)).toString(), (-1/Math.sqrt(2)).toString()];
                }
            }
            if (basis == 'Y') {
                if (n <= alphaPrR) {
                    newState.rightMeasResult = [(math.evaluate(`1/sqrt(2)`)).toString(), (math.evaluate(`i/sqrt(2)`)).toString()];
                }
                else {
                    newState.rightMeasResult = [(math.evaluate(`1/sqrt(2)`)).toString(), (math.evaluate(`-i/sqrt(2)`)).toString()];
                }
            }

            if (state.leftMeas) { 
                newState.state = [];
            }
            else {
                if (n <= alphaPrR) {
                    const normalization = '(' + (math.evaluate(`sqrt((${multByConj(newState.state[0])})+${multByConj(newState.state[2])})`)).toString() + ')';
                    newState.state = [math.evaluate(`(${newState.state[0]})/${normalization}`).toString(), math.evaluate(`(${newState.state[2]})/${normalization}`).toString()];
                }
                else {
                    const normalization = '(' + (math.evaluate(`sqrt((${multByConj(newState.state[1])})+${multByConj(newState.state[3])})`)).toString() + ')';
                    newState.state = [math.evaluate(`(${newState.state[1]})/${normalization}`).toString(), math.evaluate(`(${newState.state[3]})/${normalization}`).toString()];
                }
            }
        }
        stateArr.pop();
        stateArr.push(newState);
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
        document.getElementById("partial-density-container").appendChild(renderer.domElement);

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

        function prepareStateValToOutput(expr) {
            expr = math.format(math.evaluate(expr), 3).toString();

            expr = expr.replaceAll(" ", "");

            expr = expr.slice(0, 5);
            
            while (expr.length < 5) {
                expr = expr + " ";
            }

            return expr;
        }

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

            let labelAlice = new TextSprite({
                text: 'Alice',
                fontSize: 2,
                color: '#000000',
            });
            labelAlice.translateX(-14);
            labelAlice.translateY(25);
            scene.add(labelAlice);

            let labelBob = new TextSprite({
                text: 'Bob',
                fontSize: 2,
                color: '#000000',
            });
            labelBob.translateX(14);
            labelBob.translateY(25);
            scene.add(labelBob);
        }

        function addDensityMatrices(matricesObj) {
            if (matricesObj.reducedDensityA != undefined) {
                let densityMatrixAlice = new TextSprite({
                    text: "⎧"+matricesObj["reducedDensityA"]["0-0"]+"  "+matricesObj["reducedDensityA"]["0-1"]+"⎫|00⟩\n"+
                        "⎩"+matricesObj["reducedDensityA"]["1-0"]+"  "+matricesObj["reducedDensityA"]["1-1"]+"⎭|11⟩",
                    fontSize: 2,
                    color: '#000000',
                    fontFamily: "monospace"
                });
                densityMatrixAlice.translateX(-14);
                densityMatrixAlice.translateY(-15);
                scene.add(densityMatrixAlice);
            }
            if (matricesObj.reducedDensityB != undefined) {
                let densityMatrixBob = new TextSprite({
                    text: "⎧"+matricesObj["reducedDensityB"]["0-0"]+"  "+matricesObj["reducedDensityB"]["0-1"]+"⎫|00⟩\n"+
                        "⎩"+matricesObj["reducedDensityB"]["1-0"]+"  "+matricesObj["reducedDensityB"]["1-1"]+"⎭|11⟩",
                    fontSize: 2,
                    color: '#000000',
                    fontFamily: "monospace"
                });
                densityMatrixBob.translateX(14);
                densityMatrixBob.translateY(-15);
                scene.add(densityMatrixBob);
            }
        }

        function addMainState() {
            if (state.state.length == 4) {
                const stateLabel = new TextSprite({
                    text: "⎧"+prepareStateValToOutput(state.state[0])+"⎫|00⟩\n⎪"
                        +prepareStateValToOutput(state.state[1])+"⎪|01⟩\n⎪"
                        +prepareStateValToOutput(state.state[2])+"⎪|10⟩\n⎩"
                        +prepareStateValToOutput(state.state[3])+"⎭|11⟩",
                    fontSize: 1.5,
                    color: '#000000',
                    fontFamily: "monospace"
                });
                stateLabel.translateY(25);
                scene.add(stateLabel);
            }
            if (state.state.length == 2) {
                const stateLabel = new TextSprite({
                    text: "⎧"+prepareStateValToOutput(state.state[0])+"⎫|0⟩\n⎩"
                        +prepareStateValToOutput(state.state[1])+"⎭|1⟩",
                    fontSize: 1.5,
                    color: '#000000',
                    fontFamily: "monospace"
                });
                stateLabel.translateY(25);
                scene.add(stateLabel);
            }
        }

        function addStateToBlochSphere(densityMatrix, color, scale=0.6) {
            function addArrow(horizAngle=-Math.PI/2, vertAngle, radius=10, color=0xff0000, fromCenter=0, labelText, sideLabel = false) {
                const arrowObj = {horizAngle, vertAngle, radius, color, fromCenter, labelText};

                const lineGeometry = new THREE.CylinderGeometry(0.2*scale, 0.2*scale, (radius*2-1)*scale, 3, 1);
                const lineMaterial = new THREE.MeshStandardMaterial({color: color});
                const line = new THREE.Mesh(lineGeometry, lineMaterial);
                line.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
                line.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
                line.translateY((-0.5 + fromCenter)*scale);
                scene.add(line);
                arrowObj.line = line;

                const arrowGeometry = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
                const arrowMaterial = new THREE.MeshStandardMaterial({color: color});
                const arrow = new THREE.Mesh(arrowGeometry, arrowMaterial);
                arrow.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
                arrow.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
                arrow.translateY((radius-0.5 + fromCenter)*scale);
                scene.add(arrow);
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
                label.rotateOnWorldAxis(new THREE.Vector3(0, 0, 1), -vertAngle);
                label.rotateOnWorldAxis(new THREE.Vector3(0, 1, 0), horizAngle);
                if (sideLabel) {
                    label.translateY(fromCenter*scale);
                    label.translateX(2*scale);
                }
                else {
                    label.translateY(1.4*(radius + fromCenter)*scale);
                }
                scene.add(label);
                arrowObj.label = label;

                return arrowObj;
            }

            let arrowObj;

            const a = densityMatrix['0-0'];
            const b = densityMatrix['0-1'];
            const c = densityMatrix['1-0'];
            const d = densityMatrix['1-1'];

            let r;
            let vertAngle;
            let horizAngle;

            if (math.evaluate(a) == 0.5 && math.evaluate(b) == 0 && math.evaluate(c) == 0 && math.evaluate(d) == 0.5) {
                r = 0;
                vertAngle = 0;
                horizAngle = 0;
            }
            else {
                const z = math.evaluate(`2*(${a})-1`).toString();
                const x = math.evaluate(`(${b})+(${c})`).toString();
                const y = math.evaluate(`((${c})-(${b}))/i`).toString();
                r = math.evaluate(`sqrt((${x})^2 + (${y})^2 + (${z})^2)`);

                vertAngle = math.evaluate(`acos((${z})/(${r.toString()}))`);
                if (vertAngle == 0 || vertAngle == Math.PI) {
                    horizAngle = 0
                }
                else {
                    horizAngle = math.atan2(math.evaluate(`(${y})/((${r.toString()})*(sin(${vertAngle.toString()})))`), math.evaluate(`(${x})/((${r.toString()})*(sin(${vertAngle.toString()})))`));
                }
            }

            const alpha = math.evaluate(`cos((${vertAngle})/2)`);
            const beta = math.evaluate(`(sin((${vertAngle})/2))*e^(i*(${horizAngle}))`);

            const toOutput = [prepareStateValToOutput(math.round(alpha, 2).toString()), prepareStateValToOutput(math.round(beta, 2).toString())];

            arrowObj = addArrow((horizAngle - Math.PI/2 + 4*Math.PI)%(2*Math.PI), vertAngle, 10*r, color, 0, "⎧"+toOutput[0]+"⎫\n⎩"+toOutput[1]+"⎭");

            return arrowObj;
        }

        function addBlochSphere(scale=0.6) {
            const blochSphereObj = {};

            const circleGeometryXY = new THREE.TorusGeometry(10*scale, 0.1, 3, 100, 2*Math.PI);
            const circleMaterialXY = new THREE.MeshStandardMaterial({color: 0x000000});
            const circleXY = new THREE.Mesh(circleGeometryXY, circleMaterialXY);
            scene.add(circleXY);
            blochSphereObj.circleXY = circleXY;

            const circleGeometryXZ = new THREE.TorusGeometry(10*scale, 0.1, 3, 100, 2*Math.PI);
            const circleMaterialXZ = new THREE.MeshStandardMaterial({color: 0x000000});
            const circleXZ = new THREE.Mesh(circleGeometryXZ, circleMaterialXZ);
            circleXZ.rotateY(Math.PI/2);
            scene.add(circleXZ);
            blochSphereObj.circleXZ = circleXZ;

            const circleGeometryYZ = new THREE.TorusGeometry(10*scale, 0.1, 3, 100, 2*Math.PI);
            const circleMaterialYZ = new THREE.MeshStandardMaterial({color: 0x000000});
            const circleYZ = new THREE.Mesh(circleGeometryYZ, circleMaterialYZ);
            circleYZ.rotateX(Math.PI/2);
            scene.add(circleYZ);
            blochSphereObj.circleYZ = circleYZ;

            const lineGeometryX = new THREE.CylinderGeometry(0.1*scale, 0.1*scale, 24*scale, 3, 1);
            const lineMaterialX = new THREE.MeshStandardMaterial({color: 0x000000});
            const lineX = new THREE.Mesh(lineGeometryX, lineMaterialX);
            lineX.rotateZ(Math.PI/2);
            scene.add(lineX);
            blochSphereObj.lineX = lineX;

            const arrowGeometryX0 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
            const arrowMaterialX0 = new THREE.MeshStandardMaterial({color: 0x000000});
            const arrowX0 = new THREE.Mesh(arrowGeometryX0, arrowMaterialX0);
            arrowX0.translateX(12*scale);
            arrowX0.rotateZ(-Math.PI/2);
            scene.add(arrowX0);
            blochSphereObj.arrowX0 = arrowX0;

            const arrowGeometryX1 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
            const arrowMaterialX1 = new THREE.MeshStandardMaterial({color: 0x000000});
            const arrowX1 = new THREE.Mesh(arrowGeometryX1, arrowMaterialX1);
            arrowX1.translateX(-12*scale);
            arrowX1.rotateZ(Math.PI/2);
            scene.add(arrowX1);
            blochSphereObj.arrowX1 = arrowX1;

            const lineGeometryY = new THREE.CylinderGeometry(0.1*scale, 0.1*scale, 24*scale, 3, 1);
            const lineMaterialY = new THREE.MeshStandardMaterial({color: 0x000000});
            const lineY = new THREE.Mesh(lineGeometryY, lineMaterialY);
            scene.add(lineY);
            blochSphereObj.lineY = lineY;

            const arrowGeometryY0 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
            const arrowMaterialY0 = new THREE.MeshStandardMaterial({color: 0x000000});
            const arrowY0 = new THREE.Mesh(arrowGeometryY0, arrowMaterialY0);
            arrowY0.translateY(12*scale);
            scene.add(arrowY0);
            blochSphereObj.arrowY0 = arrowY0;

            const arrowGeometryY1 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
            const arrowMaterialY1 = new THREE.MeshStandardMaterial({color: 0x000000});
            const arrowY1 = new THREE.Mesh(arrowGeometryY1, arrowMaterialY1);
            arrowY1.translateY(-12*scale);
            arrowY1.rotateX(Math.PI);
            scene.add(arrowY1);
            blochSphereObj.arrowY1 = arrowY1;

            const lineGeometryZ = new THREE.CylinderGeometry(0.1*scale, 0.1*scale, 24*scale, 3, 1);
            const lineMaterialZ = new THREE.MeshStandardMaterial({color: 0x000000});
            const lineZ = new THREE.Mesh(lineGeometryZ, lineMaterialZ);
            lineZ.rotateX(Math.PI/2);
            scene.add(lineZ);
            blochSphereObj.lineZ = lineZ;

            const arrowGeometryZ0 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
            const arrowMaterialZ0 = new THREE.MeshStandardMaterial({color: 0x000000});
            const arrowZ0 = new THREE.Mesh(arrowGeometryZ0, arrowMaterialZ0);
            arrowZ0.translateZ(12*scale);
            arrowZ0.rotateX(Math.PI/2);
            scene.add(arrowZ0);
            blochSphereObj.arrowZ0 = arrowZ0;

            const arrowGeometryZ1 = new THREE.ConeGeometry(0.4*scale, 1*scale, 10, 1);
            const arrowMaterialZ1 = new THREE.MeshStandardMaterial({color: 0x000000});
            const arrowZ1 = new THREE.Mesh(arrowGeometryZ1, arrowMaterialZ1);
            arrowZ1.translateZ(-12*scale);
            arrowZ1.rotateX(-Math.PI/2);
            scene.add(arrowZ1);
            blochSphereObj.arrowZ1 = arrowZ1;

            let labelX0 = new TextSprite({
                text: '|𝑖⟩',
                fontSize: 2*scale,
                color: '#000000',
            });
            labelX0.translateX(14*scale);
            scene.add(labelX0);
            blochSphereObj.labelX0 = labelX0;
            
            let labelX1 = new TextSprite({
                text: '|-𝑖⟩',
                fontSize: 2*scale,
                color: '#000000',
            });
            labelX1.translateX(-14*scale);
            scene.add(labelX1);
            blochSphereObj.labelX1 = labelX1;

            let labelY0 = new TextSprite({
                text: '|0⟩',
                fontSize: 2*scale,
                color: '#000000',
            });
            labelY0.translateY(14*scale);
            scene.add(labelY0);
            blochSphereObj.labelY0 = labelY0;

            let labelY1 = new TextSprite({
                text: '|1⟩',
                fontSize: 2*scale,
                color: '#000000',
            });
            labelY1.translateY(-14*scale);
            scene.add(labelY1);
            blochSphereObj.labelY1 = labelY1;

            let labelZ0 = new TextSprite({
                text: '|+⟩',
                fontSize: 2*scale,
                color: '#000000',
            });
            labelZ0.translateZ(14*scale);
            scene.add(labelZ0);
            blochSphereObj.labelZ0 = labelZ0;
            
            let labelZ1 = new TextSprite({
                text: '|-⟩',
                fontSize: 2*scale,
                color: '#000000',
            });
            labelZ1.translateZ(-14*scale);
            scene.add(labelZ1);
            blochSphereObj.labelZ1 = labelZ1;

            return blochSphereObj;
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

        function moveGroups(objArr, x, y, z) {
            for (let obj of objArr) {
                for (let key of Object.keys(obj)) {
                    if (obj[key] && (obj[key].type == "Mesh" || obj[key].type == "Sprite")) {
                        obj[key].applyMatrix4(new THREE.Matrix4().makeTranslation(x, y, z));
                    }
                }
            }
        }

        function createSquareFrame(pos={x: 0, y: 0, z: 0}, scaleY=1, outerSize=55, thickness=1) {
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
            squareFrame.scale.y = scaleY;
            scene.add(squareFrame);

            return squareFrame;
        }

        function renderSceneElements() {
            if (stateArr[stateArr.length - 1] == state) {
                return;
            }
            scene.clear();
            state = stateArr[stateArr.length - 1];
            initScene();
            addMainState();
            let matricesObj = getDensityMatrices(state)
            if (inputObj.forgetAlice) {
                if (!state.rightMeas) {
                    matricesObj["reducedDensityB"] = reducedDensityB;
                }
                else {
                    matricesObj["reducedDensityB"] = undefined;
                }
            }
            else {
                reducedDensityB = matricesObj["reducedDensityB"];
            }
            if (inputObj.forgetBob) {
                if (!state.leftMeas) {
                    matricesObj["reducedDensityA"] = reducedDensityA;
                }
                else {
                    matricesObj["reducedDensityA"] = undefined;
                }
            }
            else {
                reducedDensityA = matricesObj["reducedDensityA"];
            }
            addDensityMatrices(matricesObj);

            const result = reverseTensorProduct(state.state);
            if (result.leftState) {
                const blochSphereObj = addBlochSphere();
                const arrowObj = addStateToBlochSphere(matricesObj["reducedDensityA"]);
                rotateGroups([blochSphereObj, arrowObj], [0, 1, 0], Math.PI/2);
                moveGroups([blochSphereObj, arrowObj], -16, 10, 0);
            }
            if (result.rightState) {
                const blochSphereObj = addBlochSphere();
                const arrowObj = addStateToBlochSphere(matricesObj["reducedDensityB"]);
                rotateGroups([blochSphereObj, arrowObj], [0, 1, 0], Math.PI/2);
                moveGroups([blochSphereObj, arrowObj], 16, 10, 0);
            }

            if (state.leftMeasResult) {
                let tempReducedDensityA;
                if (inputObj.forgetBob && !state.leftMeas) {
                    tempReducedDensityA = reducedDensityA;
                }
                else {
                    tempReducedDensityA = reducedDensityMatrixForSingeQubit(state.leftMeasResult);
                }
                const arrowObj = addStateToBlochSphere(tempReducedDensityA);
                rotateGroups([arrowObj], [0, 1, 0], Math.PI/2);
                moveGroups([arrowObj], -16, 10, 0);
            }

            if (state.rightMeasResult) {
                let tempReducedDensityB;
                if (inputObj.forgetAlice && !state.rightMeas) {
                    tempReducedDensityB = reducedDensityB;
                }
                else {
                    tempReducedDensityB = reducedDensityMatrixForSingeQubit(state.rightMeasResult);
                }
                const arrowObj = addStateToBlochSphere(tempReducedDensityB);
                rotateGroups([arrowObj], [0, 1, 0], Math.PI/2);
                moveGroups([arrowObj], 16, 10, 0);
            }

            if (!result.leftState && !result.rightState && !(state.leftMeas && state.rightMeas)) {
                const blochSphereObjL = addBlochSphere();
                const arrowObjL = addStateToBlochSphere(matricesObj["reducedDensityA"]);
                rotateGroups([blochSphereObjL, arrowObjL], [0, 1, 0], Math.PI/2);
                moveGroups([blochSphereObjL, arrowObjL], -16, 10, 0);

                const blochSphereObjR = addBlochSphere();
                const arrowObjR = addStateToBlochSphere(matricesObj["reducedDensityB"]);
                rotateGroups([blochSphereObjR, arrowObjR], [0, 1, 0], Math.PI/2);
                moveGroups([blochSphereObjR, arrowObjR], 16, 10, 0);

                createSquareFrame({x: 0, y: 10, z: 0}, 3/8);
            }
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

        function animate() {
            renderSceneElements();
            renderer.clear();
            renderer.render(backgroundScene, camera);
            renderer.render(scene, camera);
        }

        renderer.setAnimationLoop(animate);
    });
</script>

<div id="partial-density-container">
    <div id="controls-container-alice" class="controls-container">
        <span>Gate Matrix:</span>
        <div id="matrix-grid-container" style="grid-template-columns: repeat(2, 1fr);">
            {#each [0, 1] as row}
                {#each [0, 1] as col}
                    <input class="gate-matrix" id="gate-matrix-{row}{col}" bind:value={inputObj["matrixAlice"][`${row}-${col}`]}/>
                {/each}
            {/each}
        </div>
        <button id="apply-gate" on:click={() => applyGateClicked(false)}>Apply gate</button>
        <button id="alice-button-Z" style="margin-top: 1rem;" on:click={() => measureQubit(false, 'Z')}>Measure Alice's qubit in Z basis</button>
        <button id="alice-button-X" on:click={() => measureQubit(false, 'X')}>Measure Alice's qubit in X basis</button>
        <button id="alice-button-Y" on:click={() => measureQubit(false, 'Y')}>Measure Alice's qubit in Y basis</button>
        <label for="forget-checkbox-alice">No knowledge about Bob's qubit:</label>
        <input id="forget-checkbox-alice" type="checkbox" class="checkbox-round" bind:checked={inputObj.forgetBob}/>
        <button id="reset-button" style="margin-top: 1rem;" on:click={reset}>Reset</button>
    </div>
    <div id="controls-container-bob" class="controls-container">
        <span>Gate Matrix:</span>
        <div id="matrix-grid-container" style="grid-template-columns: repeat(2, 1fr);">
            {#each [0, 1] as row}
                {#each [0, 1] as col}
                    <input class="gate-matrix" id="gate-matrix-{row}{col}" bind:value={inputObj["matrixBob"][`${row}-${col}`]}/>
                {/each}
            {/each}
        </div>
        <button id="apply-gate" on:click={() => applyGateClicked(true)}>Apply gate</button>
        <button id="bob-button-Z" style="margin-top: 1rem;" on:click={() => measureQubit(true, 'Z')}>Measure Bob's qubit in Z basis</button>
        <button id="bob-button-X" on:click={() => measureQubit(true, 'X')}>Measure Bob's qubit in X basis</button>
        <button id="bob-button-Y" on:click={() => measureQubit(true, 'Y')}>Measure Bob's qubit in Y basis</button>
        <label for="forget-checkbox-bob">No knowledge about Alice's qubit:</label>
        <input id="forget-checkbox-bob" type="checkbox" class="checkbox-round" bind:checked={inputObj.forgetAlice}/>
    </div>
</div>

<style>
    .controls-container {
        position: absolute;
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    #controls-container-bob {
        right: 0%;
    }

    #matrix-grid-container {
        display: grid;
    }

    .gate-matrix {
        width: min(10vh, 5vw);
    }
</style>