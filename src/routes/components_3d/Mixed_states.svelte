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

    let stateArr = [{densityMatrix: {'0-0': "1", '0-1': "0", '1-0': "0", '1-1': "0"}}];

    let inputObj = {probs: "", states: "", horizAngle: 0, vertAngle: 0, radius: 100, densityMatrix: {'0-0': "1", '0-1': "0", '1-0': "0", '1-1': "0"}};

    let state = {};

    function updateInputStates() {
        try {
            const probsArr = inputObj.probs.split(';'); 
            const statesArr = inputObj.states.split(';').map((element) => JSON.parse(element));
            const newDensityMatrix = {'0-0': "0", '0-1': "0", '1-0': "0", '1-1': "0"};
            for (let i = 0; i < probsArr.length; i++) {
                newDensityMatrix['0-0'] = math.round(math.evaluate(`(${newDensityMatrix['0-0']})+((${probsArr[i]})*(${statesArr[i][0]})*conj(${statesArr[i][0]}))`), 3).toString();
                newDensityMatrix['0-1'] = math.round(math.evaluate(`(${newDensityMatrix['0-1']})+((${probsArr[i]})*(${statesArr[i][0]})*conj(${statesArr[i][1]}))`), 3).toString();
                newDensityMatrix['1-0'] = math.round(math.evaluate(`(${newDensityMatrix['1-0']})+((${probsArr[i]})*(${statesArr[i][1]})*conj(${statesArr[i][0]}))`), 3).toString();
                newDensityMatrix['1-1'] = math.round(math.evaluate(`(${newDensityMatrix['1-1']})+((${probsArr[i]})*(${statesArr[i][1]})*conj(${statesArr[i][1]}))`), 3).toString();
            }
            if (state.densityMatrix != newDensityMatrix) {
                stateArr.push({densityMatrix: JSON.parse(JSON.stringify(newDensityMatrix))});
                inputObj.densityMatrix = JSON.parse(JSON.stringify(newDensityMatrix));
            }
        } catch (err) {
        }
    }

    function updateInputCoords() {
        const newDensityMatrix = {'0-0': "0", '0-1': "0", '1-0': "0", '1-1': "0"};
        const x = math.evaluate(`((${inputObj.radius})/100)*sin((${inputObj.vertAngle})/100*pi)*cos((${inputObj.horizAngle})/50*pi)`);
        const y = math.evaluate(`((${inputObj.radius})/100)*sin((${inputObj.vertAngle})/100*pi)*sin((${inputObj.horizAngle})/50*pi)`);
        const z = math.evaluate(`((${inputObj.radius})/100)*cos((${inputObj.vertAngle})/100*pi)`);
        newDensityMatrix['0-0'] = math.round(math.evaluate(`(1+(${z.toString()}))/2`), 3).toString();
        newDensityMatrix['0-1'] = math.round(math.evaluate(`((${x.toString()})-i*(${y.toString()}))/2`), 3).toString();
        newDensityMatrix['1-0'] = math.round(math.evaluate(`((${x.toString()})+i*(${y.toString()}))/2`), 3).toString();
        newDensityMatrix['1-1'] = math.round(math.evaluate(`(1-(${z.toString()}))/2`), 3).toString();
        if (state.densityMatrix != newDensityMatrix) {
            stateArr.push({densityMatrix: JSON.parse(JSON.stringify(newDensityMatrix))});
            inputObj.densityMatrix = JSON.parse(JSON.stringify(newDensityMatrix));
        }
    }

    function updateInputMatrix() {
        stateArr.push({densityMatrix: JSON.parse(JSON.stringify(inputObj.densityMatrix))});
    }

    function reset() {
        stateArr = [{densityMatrix: {'0-0': "1", '0-1': "0", '1-0': "0", '1-1': "0"}}];

        inputObj = {probs: "", states: "", horizAngle: 0, vertAngle: 0, radius: 100, densityMatrix: {'0-0': "1", '0-1': "0", '1-0': "0", '1-1': "0"}};

        state = {};
    }

    let renderer;

    onDestroy(() => {
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
        document.getElementById("mixed-states-container").appendChild(renderer.domElement);

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
            expr = math.round(math.evaluate(expr), 1).toString();

            expr = expr.replaceAll(" ", "");

            expr = expr.slice(0, 9);
            
            while (expr.length < 9) {
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

            addBlochSphere();
        }

        function addStateToBlochSphere(densityMatrix, color, scale=1) {
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

            try {
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
                        horizAngle = inputObj.horizAngle/50*Math.PI;
                    }
                    else {
                        horizAngle = math.atan2(math.evaluate(`(${y})/((${r.toString()})*(sin(${vertAngle.toString()})))`), math.evaluate(`(${x})/((${r.toString()})*(sin(${vertAngle.toString()})))`));
                    }
                }

                if (r != 0) {
                    inputObj.vertAngle = vertAngle/Math.PI*100;
                    if (!((inputObj.horizAngle == 0 || inputObj.horizAngle == 100) && (horizAngle + 2*Math.PI)%(2*Math.PI) == 0)) {
                        inputObj.horizAngle = (horizAngle + 2*Math.PI)%(2*Math.PI)/Math.PI*50;
                    }
                    inputObj.radius = r*100;
                }

                const alpha = math.evaluate(`cos((${vertAngle})/2)`);
                const beta = math.evaluate(`(sin((${vertAngle})/2))*e^(i*(${horizAngle}))`);

                const toOutput = [prepareStateValToOutput(alpha.toString()), prepareStateValToOutput(beta.toString())];

                arrowObj = addArrow((horizAngle - Math.PI/2 + 4*Math.PI)%(2*Math.PI), vertAngle, 10*r, color, 0, "⎧"+toOutput[0]+"⎫\n⎩"+toOutput[1]+"⎭");

                return arrowObj;
            } catch(err) {}
        }

        function addBlochSphere(scale=1) {
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

        function renderSceneElements() {
            if (stateArr[stateArr.length - 1] == state) {
                return;
            }
            scene.clear();
            state = stateArr[stateArr.length - 1];
            initScene();
            addStateToBlochSphere(state.densityMatrix);
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

<div id="mixed-states-container">
    <div class="controls-container">
        <span>Probabilities:</span>
        <span>(Example: 0.1; 0.9)</span>
        <input type="text" id="probs" bind:value={inputObj.probs} on:change={updateInputStates}/>
        <span>Corresponding Qubit States:</span>
        <span>(Example: ["1/sqrt(2)", "-1/sqrt(2)"]; ["0", "1"])</span>
        <input type="text" id="states" bind:value={inputObj.states} on:change={updateInputStates}/>
        <span style="margin-top: 1rem;">OR</span>
        <div style="margin-top: 1rem;">
            <label for="vert-angle-input">θ:</label>
            <input class="range-input" type="range" id="vert-angle-input" bind:value={inputObj.vertAngle} on:input={updateInputCoords}/>
        </div>
        <div>
            <label for="horiz-angle-input">φ:</label>
            <input class="range-input" type="range" id="horiz-angle-input" bind:value={inputObj.horizAngle} on:input={updateInputCoords}/>
        </div>
        <div>
            <label for="radius-input">r:</label>
            <input class="range-input" type="range" id="radius-input" bind:value={inputObj.radius} on:input={updateInputCoords}/>
        </div>
        <span style="margin-top: 1rem;">OR</span>
        <span style="margin-top: 1rem;">Density Matrix:</span>
        <div id="matrix-grid-container">
            {#each Array.from(Array(2).keys()) as row}
                {#each Array.from(Array(2).keys()) as col}
                    <input class="gate-matrix" id="gate-matrix-{row}{col}" bind:value={inputObj.densityMatrix[`${row}-${col}`]} on:change={updateInputMatrix}/>
                {/each}
            {/each}
        </div>
        <button id="reset-button" on:click={reset}>Reset</button>
    </div>
</div>

<style>
    .controls-container {
        position: absolute;
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    #matrix-grid-container {
        display: grid;
        grid-template-columns: repeat(2, min(10vh, 5vw));
    }
</style>