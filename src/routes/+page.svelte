<script>
    import Introduction from "./components/Introduction.svelte";
    import WaterPool from "./components_3d/Water_pool.svelte";
    import BlochSphere from "./components_3d/Bloch_sphere.svelte";
    import BombDetection from "./components_3d/Bomb_detection.svelte";
    import Entanglement from "./components_3d/Entanglement.svelte";
    import MixedStates from "./components_3d/Mixed_states.svelte";
    import ChcsGame from "./components_3d/CHCS_game.svelte";
    import ReducedDensityMatrix from "./components_3d/Reduced_density_matrix.svelte";
    import NoCloning from "./components/NoCloning.svelte";
    import QuantumKeys from "./components_3d/Quantum_keys.svelte";
    import Decoherence from "./components_3d/Decoherence.svelte";
    import SchrodingerCat from "./components_3d/Schrodinger_cat.svelte";
    import DeutschJozsa from "./components_3d/DeutschJozsa.svelte";

    import pdfUrlIntroduction from './assets/pdfImages/Introduction.jpg?url';
    import pdfUrlDoubleSlit from './assets/pdfImages/DoubleSlit.jpg?url';
    import pdfUrlBlochSphere from './assets/pdfImages/BlochSphere.jpg?url';
    import pdfUrlBombDetection from './assets/pdfImages/BombDetection.jpg?url';
    import pdfUrlEntanglement from './assets/pdfImages/Entanglement.jpg?url';
    import pdfUrlMixedStates from './assets/pdfImages/MixedStates.jpg?url';
    import pdfUrlCHCSGame from './assets/pdfImages/CHCSGame.jpg?url';
    import pdfUrlReducedDensityMatrix from './assets/pdfImages/ReducedDensityMatrix.jpg?url';
    import pdfUrlNoCloning from './assets/pdfImages/NoCloning.jpg?url';
    import pdfUrlKeyDistribution from './assets/pdfImages/KeyDistribution.jpg?url';
    import pdfUrlDecoherence from './assets/pdfImages/Decoherence.jpg?url';
    import pdfUrlSchrodingerCat from './assets/pdfImages/SchrodingerCat.jpg?url';
    import pdfUrlDeutschJozsa from './assets/pdfImages/DeutschJozsa.jpg?url';

    let curPart = 0;

    const partNames = {
        0: "0. Introduction",
        1: "1. DoubleSlit",
        2: "2. BlochSphere",
        3: "3. BombDetection",
        4: "4. Entanglement",
        5: "5. MixedStates",
        6: "6. CHCSGame",
        7: "7. ReducedDensityMatrix",
        8: "8. NoCloning",
        9: "9. QuantumKeys",
        10: "10. Decoherence",
        11: "11. SchrodingerCat",
        12: "12. DeutschJozsa"
    };

    const components = {
        0: Introduction,
        1: WaterPool,
        2: BlochSphere,
        3: BombDetection,
        4: Entanglement,
        5: MixedStates,
        6: ChcsGame,
        7: ReducedDensityMatrix,
        8: NoCloning,
        9: QuantumKeys,
        10: Decoherence,
        11: SchrodingerCat,
        12: DeutschJozsa
    };

    const pdfURLs = {
        0: pdfUrlIntroduction,
        1: pdfUrlDoubleSlit,
        2: pdfUrlBlochSphere,
        3: pdfUrlBombDetection,
        4: pdfUrlEntanglement,
        5: pdfUrlMixedStates,
        6: pdfUrlCHCSGame,
        7: pdfUrlReducedDensityMatrix,
        8: pdfUrlNoCloning,
        9: pdfUrlKeyDistribution,
        10: pdfUrlDecoherence,
        11: pdfUrlSchrodingerCat,
        12: pdfUrlDeutschJozsa
    };

    function changePartClicked(i) {
        curPart = parseInt(i);
    }
</script>
<header>
    <div class="part-names-container">
        {#each Object.keys(components) as i}
            <span class="part-names">{partNames[i]}</span>
        {/each}
    </div>
    <div class="progress-container">
        <div class="progress-bar">
            <div class="progress-fill" style={"width: "+ 100*curPart/Math.max(...Object.keys(components).map(el => parseInt(el))) +"%;"}></div> <!-- Update this width dynamically -->
        </div>
        <div class="chapter-container">
            {#each Object.keys(components) as i}
                <button class={"chapter-circle" + ((parseInt(i) <= curPart) ? " completed" : "")} on:click={() => changePartClicked(i)}>
                    <span>{i}</span>
                </button>
            {/each}
        </div>
    </div>
</header>

<div id="container-3d">
    {#key curPart}
        <svelte:component this={components[curPart]} />
    {/key}  
</div>

<img src={pdfURLs[curPart]} alt="description" id="pdf-image-container"/>

<style>
    #pdf-image-container {
        border-radius: 15px;
        margin-top: 10px;
        margin-left: auto;
        width: 50%;
        background-color: black;
    }

    header {
        backdrop-filter: blur(100px);
        position: absolute;
        width: 100%;
        background-color: rgba(20, 25, 30, 0.1);
        -webkit-mask-image: linear-gradient(to bottom, 
                                rgb(0, 0, 0) 10vh, 
                                transparent);
        mask-image: linear-gradient(to bottom, 
                                rgb(0, 0, 0) 10vh, 
                                transparent);
        text-align: center;
        height: 13vh;
        align-content: center;
    }

    .progress-container {
        position: relative;
        width: 90%;
        margin: auto;
    }

    .progress-bar {
        box-shadow: inset 0px 0px 5px 0px rgba(202, 202, 202, 0.99);
        position: absolute;
        top: 50%;
        transform: translateY(-50%);
        width: 97%;
        margin-left: 1.5%;
        height: 1.5vh;
        background-color: rgba(255, 255, 255, 0.8);
        z-index: 0;
    }

    .progress-fill {
        box-shadow: 0px 0px 5px 2px #a6a6ff;
        height: 100%;
        background-color: rgba(255, 255, 255, 0.8);
        z-index: 1;
    }

    .chapter-container {
        display: flex;
        justify-content: space-between;
        position: relative;
        z-index: 2;
    }

    .part-names-container {
        display: flex;
        justify-content: space-between;
        position: relative;
        z-index: 2;
        margin: 1vh auto;
        width: 90%;
        color: rgb(0, 0, 0);
    }

    .chapter-circle {
        padding: 0;
        border-style: none;
        box-shadow: inset 0px 0px 5px 0px rgba(202, 202, 202, 0.99);
        width: 4.5vh;
        height: 4.5vh;
        border-radius: 50%;
        background-color: white;
        position: relative;
        z-index: 3; /* Ensure circles are above the progress bar */
    }

    .completed {
        box-shadow: 0px 0px 5px 2px #a6a6ff;
        background-color: white;
    }
</style>