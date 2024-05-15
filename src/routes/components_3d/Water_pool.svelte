<script>
  import { pauseStore } from './store.js';
  import WaterPoolWithoutControls from "./Water_pool_without_controls.svelte";

  let place_photon = true; // Allows to play with the water without any quantum mechanics attached.
  let slit_mode = '0'; // 0 - two slits, 1 - one slit, 2 - one measurement apparatus and one slit. 
  let remove_water = false; // Allows users to see the interactions of amplitudes more clearly.
  let prob_pattern = false; // Diffraction/probability pattern.
  let brightness_increase = false; // Makes photons brighten the scene. Produces lags because the
  let remove_photons = true;
  let allow_many = true;

  function pauseButtonClicked() {
    pauseStore.update((prevVal) => !prevVal)
  }

  let varToForceUpdate = false;

  function changeVarToForceUpdate() {
    varToForceUpdate = false;
  }
</script>

<div id="water-pool-container-with-controls">
  <div class="controls-container">
    <div>
      <label for="place-photon-checkbox">Place photon on click</label>
      <input type="checkbox" id="place-photon-checkbox" class="checkbox-round" bind:checked={place_photon} on:change={() => varToForceUpdate = true}/>
    </div>
    <div>
      <label for="remove-water-checkbox">Remove water</label>
      <input type="checkbox" id="remove-water-checkbox" class="checkbox-round" bind:checked={remove_water} on:change={() => varToForceUpdate = true}/>
    </div>
    <div>
      <label for="interference-checkbox">Display interference pattern on the walls</label>
      <input type="checkbox" id="interference-checkbox" class="checkbox-round" bind:checked={prob_pattern} on:change={() => varToForceUpdate = true}/>
    </div>
    <div>
      <label for="brightness-checkbox">Photons brighten the walls</label>
      <input type="checkbox" id="brightness-checkbox" class="checkbox-round" bind:checked={brightness_increase} disabled={!prob_pattern} on:change={() => varToForceUpdate = true}/>
    </div>
    <div>
      <label for="remove-photons-checkbox">Remove photons upon reaching the wall</label>
      <input type="checkbox" id="remove-photons-checkbox" class="checkbox-round" bind:checked={remove_photons} on:change={() => varToForceUpdate = true}/>
    </div>
    <div>
      <label for="allow-many-checkbox">Allow many photons</label>
      <input type="checkbox" id="allow-many-checkbox" class="checkbox-round" bind:checked={allow_many} on:change={() => varToForceUpdate = true}/>
    </div>
    <div>
      <label for="slit-mode-options">Choose an option:</label>
      <select id="slit-mode-options" name="options" bind:value={slit_mode} on:change={() => varToForceUpdate = true}>
          <option value="0">Two open slits</option>
          <option value="1">One open slit</option>
          <option value="2">One slit with measurement</option>
      </select>
    </div>
    <button id="pause-button" on:click={pauseButtonClicked}>Pause Animation</button>
  </div>
  {#if varToForceUpdate}
    {changeVarToForceUpdate()}
  {:else}
    <WaterPoolWithoutControls {place_photon} {slit_mode} {remove_water} {prob_pattern} {brightness_increase} {remove_photons} {allow_many}/>
  {/if}
</div>

<style>
  .controls-container {
      position: absolute;
      display: flex;
      flex-direction: column;
      align-items: center;
  }
</style>