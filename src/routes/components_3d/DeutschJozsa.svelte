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
    import { SceneNode } from "three/examples/jsm/nodes/Nodes.js";

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
    
    let state = {scene: undefined, waves: [], numWaves: 0, fLabel: undefined};

    let demoStep = 0;

    let inputObj = {quantum: true, balanced: true, f00: 0, f01: 1, f10: 0, f11: 1, x: "00"};

    let a00;
    let a01;
    let a10;
    let a11;

    function reset() {
        state = {scene: state.scene, waves: [], numWaves: 0, fLabel: undefined};

        demoStep = 0;

        inputObj = {quantum: true, balanced: true, f00: 0, f01: 1, f10: 0, f11: 1, x: "00"};

        a00;
        a01;
        a10;
        a11;

        changeScene();
    }

    function checkInputF() {
        inputObj.f00 = (Math.floor(inputObj.f00) + 2) % 2;
        inputObj.f01 = (Math.floor(inputObj.f01) + 2) % 2;
        inputObj.f10 = (Math.floor(inputObj.f10) + 2) % 2;
        inputObj.f11 = (Math.floor(inputObj.f11) + 2) % 2;

        state.scene.remove(state.fLabel);
        state.fLabel = addLabel(`⎧${getFMatrixValue(inputObj.f00)}  0  0  0⎫\n⎪ 0 ${getFMatrixValue(inputObj.f01)}  0  0⎪\n⎪ 0  0 ${getFMatrixValue(inputObj.f10)}  0⎪\n⎩ 0  0  0 ${getFMatrixValue(inputObj.f11)}⎭`, 4, -17.5, "#0000ff");
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
    }

    function changeScene() {
        state.scene.clear();
        initScene();
        initDemoScene();
    }

    function getFMatrixValue(fVal) {
        if (fVal == 0) {
            return " 1";
        }
        else {
            return "-1";
        }
    }

    function prepareStateValToOutput(expr) {
        expr = math.format(math.evaluate(expr), 2).toString();

        expr = expr.replaceAll(" ", "");

        expr = expr.slice(0, 5);
        
        while (expr.length < 4) {
            expr = expr + " ";
        }

        return expr;
    }

    function nextStep() {
        demoStep++;
        if (demoStep == 1) {
            addWave(-33.5, 10, -Math.PI/2, "|0⟩", 0x00ffff);
            addWave(-33.5, 5, -Math.PI/2, "|0⟩", 0x00ffff);
            addLabel("⎧1⎫|0⟩\n⎩0⎭|1⟩\n\n⎧1⎫|0⟩\n⎩0⎭|1⟩", -47, -17.5);
        }
        if (demoStep == 2) {
            addWave(-23.5, 0, -Math.PI/2, "|0⟩", 0x00ffff);
            addWave(-23.5, 5, -Math.PI/2, "|1⟩", 0xffffff);
            addWave(-23.5, 10, -Math.PI/2, "|0⟩", 0x00ffff);
            addWave(-23.5, 15, -Math.PI/2, "|1⟩", 0xffffff);
            addLabel("⎧1/√2⎫|0⟩\n⎩1/√2⎭|1⟩\n\n⎧1/√2⎫|0⟩\n⎩1/√2⎭|1⟩", -23, -17.5); //⎧1⎫⎪0⎪⎩1⎭
        }
        if (demoStep == 3) {
            addWave(-13.5, 0, -Math.PI/2, "|00⟩", 0x00ffff);
            addWave(-13.5, 5, -Math.PI/2, "|01⟩", 0x00ffff);
            addWave(-13.5, 10, -Math.PI/2, "|10⟩", 0xffffff);
            addWave(-13.5, 15, -Math.PI/2, "|11⟩", 0xffffff);
            addWave(-13.9, 0, -Math.PI/2, "", 0x00ffff);
            addWave(-13.9, 5, -Math.PI/2, "", 0xffffff);
            addWave(-13.9, 10, -Math.PI/2, "", 0x00ffff);
            addWave(-13.9, 15, -Math.PI/2, "", 0xffffff);
            addLabel("1/2", -14, -17.5);
            addLabel("⎧1⎫|00⟩\n⎪1⎪|01⟩\n⎪1⎪|10⟩\n⎩1⎭|11⟩", -8, -17.5); //⎧1⎫⎪0⎪⎩1⎭
        }
        if (demoStep == 4) {
            if (inputObj.f00 == 0) {
                addWave(-3.5, 0, -Math.PI/2, "|1⟩", 0xffffff, 28);
            }
            else {
                addWave(-3.5, 0, -Math.PI/2, "|-1⟩", 0x000000, 28);
            }
            if (inputObj.f01 == 0) {
                addWave(-3.5, 5, -Math.PI/2, "|1⟩", 0xffffff, 10);
            }
            else {
                addWave(-3.5, 5, -Math.PI/2, "|-1⟩", 0x000000, 10);
            }
            if (inputObj.f10 == 0) {
                addWave(-3.5, 10, -Math.PI/2, "|1⟩", 0xffffff, 10);
            }
            else {
                addWave(-3.5, 10, -Math.PI/2, "|-1⟩", 0x000000, 10);
            }
            if (inputObj.f11 == 0) {
                addWave(-3.5, 15, -Math.PI/2, "|1⟩", 0xffffff, 28);
            }
            else {
                addWave(-3.5, 15, -Math.PI/2, "|-1⟩", 0x000000, 28);
            }

            addLabel("1/2", 13, -17.5);
            a00 = getFMatrixValue(inputObj.f00);
            a01 = getFMatrixValue(inputObj.f01);
            a10 = getFMatrixValue(inputObj.f10);
            a11 = getFMatrixValue(inputObj.f11);
            addLabel(`⎧${a00}⎫|00⟩\n⎪${a01}⎪|01⟩\n⎪${a10}⎪|10⟩\n⎩${a11}⎭|11⟩`, 19, -17.5); //⎧1⎫⎪0⎪⎩1⎭
        }
        if (demoStep == 5) {
            function applyTransformTwoQubits(matrix=[['1/2', '1/2', '1/2', '1/2'], ['1/2', '-1/2', '1/2', '-1/2'], ['1/2', '1/2', '-1/2', '-1/2'], ['1/2', '-1/2', '-1/2', '1/2']]) {
                const s = [a00+'/2', a01+'/2', a10+'/2', a11+'/2'];
                let newState = [];
                for (let i = 0; i < 4; i++) {
                    newState.push((math.evaluate(
                    "(("+matrix[i][0]+')*('+s[0]+"))+\
                        (("+matrix[i][1]+')*('+s[1]+"))+\
                        (("+matrix[i][2]+')*('+s[2]+"))+\
                        (("+matrix[i][3]+')*('+s[3]+"))"
                    )).toString());
                }
                a00 = prepareStateValToOutput(newState[0]);
                a01 = prepareStateValToOutput(newState[1]);
                a10 = prepareStateValToOutput(newState[2]);
                a11 = prepareStateValToOutput(newState[3]);
            }
            if ((inputObj.f00 + inputObj.f01 + inputObj.f10 + inputObj.f11)%2 == 0) {
                createBox(4, 4, Math.PI/4, 6, 7.5, '', inputObj.balanced ? 0x000000 : 0xffffff, 1);
                const label = addLabel(inputObj.balanced ? "0" : "1", 6, 7.5, inputObj.balanced ? "#ffffff" : "#000000", 7);
                label.translateZ(1);
            }

            applyTransformTwoQubits();
            addLabel(`⎧${a00}⎫|00⟩\n⎪${a01}⎪|01⟩\n⎪${a10}⎪|10⟩\n⎩${a11}⎭|11⟩`, 51, -17.5);
            if (Math.random() <= math.evaluate(`abs(${a00})`)) {
                addLabel('1', 63, -17.5, "#000000", 7);
            }
            else {
                addLabel('0', 63, -17.5, "#000000", 7);
            }
    }
    }

    function addLabel(text, x, y, color="#000000", fontSize=2) {
        const label = new TextSprite({
            text: text,
            fontSize: fontSize,
            fontFamily: "monospace",
            color: color,
        });
        label.translateX(x);
        label.translateY(y);
        label.material.depthTest = false;
        label.material.needsUpdate = true;
        state.scene.add(label);

        return label;
    }

    function createBox(length, height, rotate, x, y, label="", color=0x0000aa, opacity=0.2) {
        const boxGeometry = new THREE.BoxGeometry(length, height, length, 1, 1, 1);
        const boxMaterial = new THREE.MeshStandardMaterial({transparent: true, opacity: opacity, color: color, side: THREE.DoubleSide});
        const box = new THREE.Mesh(boxGeometry, boxMaterial);
        box.translateY(y);
        box.translateX(x);
        box.rotateZ(rotate);
        state.scene.add(box);

        addLabel(label, x, y);
    }

    function initDemoScene() {
        function createChannel(length, rotate, x, y, label="") {
            const channelGeometry = new THREE.CylinderGeometry(1, 1, length, 20, 1, true);
            const channelMaterial = new THREE.MeshStandardMaterial({transparent: true, opacity: 0.2, color: 0x000000, side: THREE.DoubleSide});
            const channel = new THREE.Mesh(channelGeometry, channelMaterial);
            channel.translateY(y);
            channel.translateX(x);
            channel.rotateZ(rotate);
            state.scene.add(channel);

            addLabel(label, x, y);
        }

        if (inputObj.quantum) {
            addLabel("Demonstration of interference between the classical outcomes (Not a circuit)", 0, 20);

            createChannel(7, Math.PI/2, -30, 10);
            createChannel(7, Math.PI/2, -30, 5);

            createBox(3, 8, 0, -25, 12.5, "H");
            createBox(3, 8, 0, -25, 2.5, "H");

            createChannel(7, Math.PI/2, -20, 15);
            createChannel(7, Math.PI/2, -20, 10);
            createChannel(7, Math.PI/2, -20, 5);
            createChannel(7, Math.PI/2, -20, 0);

            createBox(3, 20, 0, -15, 7.5, "⊗");

            createChannel(7, Math.PI/2, -10, 15);
            createChannel(7, Math.PI/2, -10, 10);
            createChannel(7, Math.PI/2, -10, 5);
            createChannel(7, Math.PI/2, -10, 0);

            createBox(3, 4, 0, -5, 15, "Uf");
            createBox(3, 4, 0, -5, 10, "Uf");
            createBox(3, 4, 0, -5, 5, "Uf");
            createBox(3, 4, 0, -5, 0, "Uf");

            createChannel(18, Math.PI/2, 5, 15);
            createChannel(7, Math.PI/2, 0, 10);
            createChannel(7, Math.PI/2, 0, 5);
            createChannel(18, Math.PI/2, 5, 0);

            createChannel(10, -Math.PI/4, 10, 11.8);
            createChannel(3, Math.PI/4, 4, 9.3);
            createChannel(3, -Math.PI/4, 4, 5.7);
            createChannel(10, Math.PI/4, 10, 3.2);

            createBox(3, 3, Math.PI/4, 6, 7.5, "H⊗H");

            addLabel("Math Operations (To show how we get the amplitudes for each outcome above. Matrices are applied on the left.)", 0, -5);

            addLabel("⎧1/√2  1/√2⎫\n⎩1/√2 -1/√2⎭\n\n⎧1/√2  1/√2⎫\n⎩1/√2 -1/√2⎭", -35, -17.5, "#0000ff");

            addLabel("⊗", -17, -17.5, "#0000ff");

            state.fLabel = addLabel(`⎧${getFMatrixValue(inputObj.f00)}  0  0  0⎫\n⎪ 0 ${getFMatrixValue(inputObj.f01)}  0  0⎪\n⎪ 0  0 ${getFMatrixValue(inputObj.f10)}  0⎪\n⎩ 0  0  0 ${getFMatrixValue(inputObj.f11)}⎭`, 4, -17.5, "#0000ff");

            addLabel("⎧1/2  1/2  1/2  1/2⎫\n⎪1/2 -1/2  1/2 -1/2⎪\n⎪1/2  1/2 -1/2 -1/2⎪\n⎩1/2 -1/2 -1/2  1/2⎭", 35, -17.5, "#0000ff"); //⎧⎫⎪⎪⎩⎭
            addLabel("⟨00|", 59, -17.5, "#0000ff")
        }
        else {
            addLabel(`f(${inputObj.x}) = ${inputObj[`f${inputObj.x}`]}`, 0, 0);
        }

    }

    function addWave(x=0, y=0, direction=0, text="", color=0x000000, wayLength=7) {
        const waveObj = {x, y, direction, text, color, wayLength, wayComplition: 0};

        const points = [];
        const depth = 0.25; // Thickness of the lens
        const radius = 1;  // Radius of the lens at the widest point

        // Define the shape of one side of the lens
        for (let i = Math.PI/2; i <= Math.PI; i += Math.PI / 10) {
            let xIn;
            let yIn;
            if (i == Math.PI) {
                xIn = 0
                yIn = depth+0.01;
            }
            else {
                xIn = Math.sin(i) * radius;
                yIn = (i - Math.PI / 2) * depth;
            }
            points.push(new THREE.Vector2(xIn, yIn));
        }

        const coneGeometry = new THREE.LatheGeometry(points, 40);
        const coneMaterial = new THREE.MeshStandardMaterial({color: color, side: THREE.DoubleSide});
        const cone = new THREE.Mesh(coneGeometry, coneMaterial);
        cone.translateY(y);
        cone.translateX(x);
        cone.rotateZ(direction);
        waveObj.cone = cone;
        state.scene.add(cone);

        waveObj.label = addLabel(text, x, y, "#ff0000");
        state.waves.push(waveObj);
        state.numWaves++;

        return waveObj;
    }

    function moveWavesOneStep(stepSize=0.1) {
        for (let i = state.waves.length - 1; i >= 0; i--) {
            let waveObj = state.waves[i];
            let cone = waveObj.cone;
            let label = waveObj.label;

            if (waveObj.wayComplition >= 1) {
                state.scene.remove(cone);
                state.scene.remove(label);
                state.waves.splice(i, 1);
                state.numWaves--;
            }

            if (demoStep == 4 && waveObj.y == 15 && waveObj.wayComplition >= 17/28 && waveObj.direction == -Math.PI/2) {
                state.scene.remove(cone);
                state.scene.remove(label);
                state.waves.splice(i, 1);
                state.numWaves--;

                waveObj = addWave(waveObj.x+17, waveObj.y, 3*Math.PI/4, waveObj.text, waveObj.color, 28);
                waveObj.wayComplition = 18/28;
                cone = waveObj.cone;
                label = waveObj.label;
            }

            if (demoStep == 4 && waveObj.y == 0 && waveObj.wayComplition >= 17/28 && waveObj.direction == -Math.PI/2) {
                state.scene.remove(cone);
                state.scene.remove(label);
                state.waves.splice(i, 1);
                state.numWaves--;

                waveObj = addWave(waveObj.x+17, waveObj.y, Math.PI/4, waveObj.text, waveObj.color, 28);
                waveObj.wayComplition = 18/28;
                cone = waveObj.cone;
                label = waveObj.label;
            }

            if (demoStep == 4 && waveObj.y == 5 && waveObj.wayComplition >= 7/10 && waveObj.direction == -Math.PI/2) {
                state.scene.remove(cone);
                state.scene.remove(label);
                state.waves.splice(i, 1);
                state.numWaves--;

                waveObj = addWave(waveObj.x+7, waveObj.y, -1*Math.PI/4, waveObj.text, waveObj.color, 10);
                waveObj.wayComplition = 7/10;
                cone = waveObj.cone;
                label = waveObj.label;
            }

            if (demoStep == 4 && waveObj.y == 10 && waveObj.wayComplition >= 7/10 && waveObj.direction == -Math.PI/2) {
                state.scene.remove(cone);
                state.scene.remove(label);
                state.waves.splice(i, 1);
                state.numWaves--;

                waveObj = addWave(waveObj.x+7, waveObj.y, 5*Math.PI/4, waveObj.text, waveObj.color, 10);
                waveObj.wayComplition = 7/10;
                cone = waveObj.cone;
                label = waveObj.label;
            }

            cone.applyMatrix4(new THREE.Matrix4().makeTranslation(-Math.sin(waveObj.direction)*waveObj.wayLength*stepSize, 
                                Math.cos(waveObj.direction)*waveObj.wayLength*stepSize, 0));
            label.applyMatrix4(new THREE.Matrix4().makeTranslation(-Math.sin(waveObj.direction)*waveObj.wayLength*stepSize, 
                                Math.cos(waveObj.direction)*waveObj.wayLength*stepSize, 0));

            waveObj.wayComplition += stepSize;

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
        document.getElementById("deutsch-joza-container").appendChild(renderer.domElement);

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
        initDemoScene();

        function animate() {
            moveWavesOneStep(0.01);
            renderer.clear();
            renderer.render(backgroundScene, camera);
            renderer.render(scene, camera);
        }

        renderer.setAnimationLoop(animate);
    });
</script>

<div id="deutsch-joza-container">
    <div class="controls-container">
        <div>
            <label for="quantum-sheckbox">Quantum</label>
            <input type="checkbox" class="checkbox-round" id="quantum-checkbox" bind:checked={inputObj.quantum} on:change={changeScene}/>
        </div>
        <div>
            <label for="balanced-sheckbox">Balanced</label>
            <input type="checkbox" class="checkbox-round" id="balanced-checkbox" bind:checked={inputObj.balanced} disabled={demoStep>0}/>
        </div>
        <div>
            <label for="f00-input">f(00)=</label>
            <input type="number" id="f00-input" bind:value={inputObj.f00} on:change={checkInputF} disabled={demoStep>0}/>
        </div>
        <div>
            <label for="f01-input">f(01)=</label>
            <input type="number" id="f01-input" bind:value={inputObj.f01} on:change={checkInputF} disabled={demoStep>0}/>
        </div>
        <div>
            <label for="f10-input">f(10)=</label>
            <input type="number" id="f10-input" bind:value={inputObj.f10} on:change={checkInputF} disabled={demoStep>0}/>
        </div>
        <div>
            <label for="f11-input">f(11)=</label>
            <input type="number" id="f11-input" bind:value={inputObj.f11} on:change={checkInputF} disabled={demoStep>0}/>
        </div>
        {#if (inputObj.balanced && (inputObj.f00 + inputObj.f01 + inputObj.f10 + inputObj.f11 != 2))}
            <span>This is not a balanced function!!!</span>
            <span>Number of 0's should be equal to number of 1's</span>
        {/if}
        {#if (!inputObj.balanced && ((inputObj.f00 + inputObj.f01 + inputObj.f10 + inputObj.f11) % 4 != 0))}
            <span>This is not a constant function!!!</span>
            <span>All output values should be only 0's or only 1's</span>
        {/if}
        {#if !inputObj.quantum}
            <div>
                <label for="x-input">Your x:</label>
                <input id="x-input" bind:value={inputObj.x} on:change={changeScene}/>
            </div>
        {:else}
            <button id="start-button" style="background-color:rgba(160, 255, 160);" on:click={nextStep} disabled={state.numWaves > 0 || demoStep > 4}>Next Step</button>
        {/if}
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