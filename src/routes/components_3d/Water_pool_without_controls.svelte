<script>
    import { afterUpdate, onDestroy } from "svelte";
    import { pauseStore } from './store.js';
    import * as THREE from "three";
    import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
    import tiles from "../assets/textures/tiles.jpg";
    import xneg from "../assets/textures/xneg.jpg";
    import xpos from "../assets/textures/xpos.jpg";
    import ypos from "../assets/textures/ypos.jpg";
    import zneg from "../assets/textures/zneg.jpg";
    import zpos from "../assets/textures/zpos.jpg";
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
    const textureloader = new THREE.TextureLoader();
  
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
  
    // This section explains why we say "amplitudes", demonstrates wave-particle duality, aspects of measurement, etc. 
    
    // Press "P" to pause animation
    export let place_photon = true; // Allows to play with the water without any quantum mechanics attached.
    export let slit_mode = '0'; // 0 - two slits, 1 - one slit, 2 - one measurement apparatus and one slit. 
    export let remove_water = false; // Allows users to see the interactions of amplitudes more clearly.
    export let prob_pattern = false; // Diffraction/probability pattern.
    export let brightness_increase = false; // Makes photons brighten the scene. Produces lags because the code was not written with lighting updates in mind.
    export let remove_photons = true;
    export let allow_many = true;
    let pause = false;
    pauseStore.subscribe((val) => {
        pause = val;
    });
    let renderer;
  
    let prob_pattern_intensity_front;
    let prob_pattern_intensity_rear;
    
  
    let shaders_utils;
    function updateShaderUtils() {
      shaders_utils = `
        #define M_PI 3.1415926535897932384626433832795
  
        const float IOR_AIR = 1.0;
        const float IOR_WATER = 1.333;
  
        const vec3 abovewaterColor = vec3(0.25, 1.0, 1.25);
        const vec3 underwaterColor = vec3(0.4, 0.9, 1.0);
  
        const float poolHeight = 1.0;
  
        uniform vec3 light;
        uniform sampler2D tiles;
        uniform sampler2D causticTex;
        uniform sampler2D water;
  
  
        vec2 intersectSlits(vec3 origin, vec3 ray, vec3 cubeMin, vec3 cubeMax) {
        float k = (ray.z * 1000.0) / (ray.z * 1000.0 - origin.z);
        vec3 midIntersect = (origin - ray * 1000.0) * k + ray * 1000.0;
        if (((ray.z > 0.0 && origin.z < 0.0) || (ray.z < 0.0 && origin.z > 0.0)) && 
            ${(slit_mode == '0' || slit_mode == '2') ? 
              "(midIntersect.x < -0.3 || midIntersect.x > 0.3 || (midIntersect.x > -0.2 && midIntersect.x < 0.2))" 
            :
              "(midIntersect.x < -0.3 || midIntersect.x > 0.2 || (midIntersect.x > -0.2 && midIntersect.x < 0.2))"
            }
          ) {
          cubeMin = vec3(-1.0, -poolHeight, 0.0);
          cubeMax = vec3(1.0, 2.0, 0.0);
        }
        vec3 tMin = (cubeMin - origin) / ray;
        vec3 tMax = (cubeMax - origin) / ray;
        vec3 t1 = min(tMin, tMax);
        vec3 t2 = max(tMin, tMax);
        float tNear = max(max(t1.x, t1.y), t1.z);
        float tFar = min(min(t2.x, t2.y), t2.z);
        return vec2(tNear, tFar);
        }
  
        vec2 intersectCube(vec3 origin, vec3 ray, vec3 cubeMin, vec3 cubeMax) {
        vec3 tMin = (cubeMin - origin) / ray;
        vec3 tMax = (cubeMax - origin) / ray;
        vec3 t1 = min(tMin, tMax);
        vec3 t2 = max(tMin, tMax);
        float tNear = max(max(t1.x, t1.y), t1.z);
        float tFar = min(min(t2.x, t2.y), t2.z);
        return vec2(tNear, tFar);
        }
  
        vec3 getWallColor(vec3 point) {
          float scale = ${prob_pattern ? ((prob_pattern_intensity_front + prob_pattern_intensity_rear)/2).toFixed(3) : 0.5};
          ${prob_pattern ?
            (slit_mode == '0' ?
              `
                if (point.z > 0.999) {
                  float x = point.x;
                  scale = ${prob_pattern_intensity_rear.toFixed(3)}*(pow(cos(M_PI*5.0*x/sqrt(1.0+x*x)), 2.0)-abs(0.6*x));
                }
                if (point.z < -0.999) {
                  float x = point.x;
                  scale = ${prob_pattern_intensity_front.toFixed(3)}*(pow(cos(M_PI*5.0*x/sqrt(1.0+x*x)), 2.0)-abs(0.6*x));
                }
              `
            :
              `
                if (point.z > 0.999) {
                  float x = point.x + 0.25;
                  float beta = M_PI*5.0*x/sqrt(1.0+x*x);
                  scale = ${prob_pattern_intensity_rear.toFixed(3)}*(pow(sin(beta)/beta, 2.0));
                }
                if (point.z < -0.999) {
                  float x = point.x + 0.25;
                  float beta = M_PI*5.0*x/sqrt(1.0+x*x);
                  scale = ${prob_pattern_intensity_front.toFixed(3)}*(pow(sin(beta)/beta, 2.0));
                }
              `
            )
          :
            ""
          }
  
          vec3 wallColor;
          vec3 normal;
          if (abs(point.x) > 0.999) {
            wallColor = texture2D(tiles, point.yz * 0.5 + vec2(1.0, 0.5)).rgb;
            normal = vec3(-point.x, 0.0, 0.0);
          } else if (abs(point.z) > 0.999 || (point.z > -0.002 && point.z < 0.002)) {
            wallColor = texture2D(tiles, point.yx * 0.5 + vec2(1.0, 0.5)).rgb;
            normal = vec3(0.0, 0.0, -point.z);
          } else {
            wallColor = texture2D(tiles, point.xz * 0.5 + 0.5).rgb;
            normal = vec3(0.0, 1.0, 0.0);
          }
  
          /* pool ambient occlusion */
          if (abs(point.z) > 0.002) {
            scale /= length(point);
          }
          else {
            scale /= length(vec3(point.x, point.y, 1));
          }
  
          /* caustics */
          vec3 refractedLight = -refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);
          float diffuse = max(0.0, dot(refractedLight, normal));
          vec4 info = texture2D(water, point.xz * 0.5 + 0.5);
          if (point.y < info.r) {
            vec4 caustic = texture2D(causticTex, 0.75 * (point.xz - point.y * refractedLight.xz / refractedLight.y) * 0.5 + 0.5);
            scale += diffuse * caustic.r * 2.0 * caustic.g;
          } else {
            /* shadow for the rim of the pool */
            vec2 t = intersectCube(point, refractedLight, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));
            diffuse *= 1.0 / (1.0 + exp(-200.0 / (1.0 + 10.0 * (t.y - t.x)) * (point.y + refractedLight.y * t.y - 2.0 / 12.0)));
  
            scale += diffuse * 0.5;
          }
  
          return wallColor * scale;
        }
        `;
        THREE.ShaderChunk['utils'] = shaders_utils;
    }
      
    let caustics_fragment;
    function updateCausticsFragment() {
      caustics_fragment = `
        precision highp float;
        precision highp int;
  
        #extension GL_OES_standard_derivatives : enable
        
        ${shaders_utils}
      
        varying vec3 oldPos;
        varying vec3 newPos;
        varying vec3 ray;
      
      
        void main() {
          /* if the triangle gets smaller, it gets brighter, and vice versa */
          float oldArea = length(dFdx(oldPos)) * length(dFdy(oldPos));
          float newArea = length(dFdx(newPos)) * length(dFdy(newPos));
          gl_FragColor = vec4(oldArea / newArea * 0.2, 1.0, 0.0, 0.0);
      
          vec3 refractedLight = refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);
      
          /* shadow for the rim of the pool */
          vec2 t = intersectCube(newPos, -refractedLight, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));
          gl_FragColor.r *= 1.0 / (1.0 + exp(-200.0 / (1.0 + 10.0 * (t.y - t.x)) * (newPos.y - refractedLight.y * t.y - 2.0 / 12.0)));
        }`
      ;
    };
    
    let caustics_vertex; 
    function updateCausticsVertex() {
      caustics_vertex = `
        precision highp float;
        precision highp int;
      
        varying vec3 oldPos;
        varying vec3 newPos;
        varying vec3 ray;
        attribute vec3 position;
      
        ${shaders_utils}
      
      
        /* project the ray onto the plane */
        vec3 project(vec3 origin, vec3 ray, vec3 refractedLight) {
          vec2 tcube = intersectCube(origin, ray, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));
          origin += ray * tcube.y;
          float tplane = (-origin.y - 1.0) / refractedLight.y;
      
          return origin + refractedLight * tplane;
        }
      
      
        void main() {
          vec4 info = texture2D(water, position.xy * 0.5 + 0.5);
          info.ba *= 0.5;
          vec3 normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
      
          /* project the vertices along the refracted vertex ray */
          vec3 refractedLight = refract(-light, vec3(0.0, 1.0, 0.0), IOR_AIR / IOR_WATER);
          ray = refract(-light, normal, IOR_AIR / IOR_WATER);
          oldPos = project(position.xzy, refractedLight, refractedLight);
          newPos = project(position.xzy + vec3(0.0, info.r, 0.0), ray, refractedLight);
      
          gl_Position = vec4(0.75 * (newPos.xz + refractedLight.xz / refractedLight.y), 0.0, 1.0);
        }`
      ;
    };
    
    let pool_fragment;
    function updatePoolFragment() {
      pool_fragment = `
        precision highp float;
        precision highp int;
      
        ${shaders_utils}
      
        varying vec3 pos;
      
      
        void main() {
          gl_FragColor = vec4(getWallColor(pos), 1.0);
      
          vec4 info = texture2D(water, pos.xz * 0.5 + 0.5);
      
          if (pos.y < info.r) {
            gl_FragColor.rgb *= underwaterColor * 1.2;
          }
        }`
      ;
    };
    
    let pool_vertex;
    function updatePoolVertex() {
      pool_vertex = `
        ${shaders_utils}
      
        uniform mat4 projectionMatrix;
        uniform mat4 modelViewMatrix;
      
        attribute vec3 position;
      
        varying vec3 pos;
      
      
        void main() {
          pos = position.xyz;
          pos.y = ((1.0 - pos.y) * (7.0 / 12.0) - 1.0) * poolHeight;
      
          gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }`
      ;
    };
    
    let simulation_drop_fragment;
    function updateSimulationDropFragment() {
      simulation_drop_fragment = `
        precision highp float;
        precision highp int;
      
        const float PI = 3.141592653589793;
        uniform sampler2D texture;
        uniform vec2 center;
        uniform float radius;
        uniform float strength;
        varying vec2 coord;
      
      
        void main() {
          /* Get vertex info */
          vec4 info = texture2D(texture, coord);
      
          /* Add the drop to the height */
          float drop = max(0.0, 1.0 - length(center * 0.5 + 0.5 - coord) / radius);
          drop = 0.5 - cos(drop * PI) * 0.5;
          info.r += drop * strength;
      
          gl_FragColor = info;
        }`
      ;
    };
    
    let simulation_normal_fragment;
    function updateSimulationNormalFragment() {
      simulation_normal_fragment = `
        precision highp float;
        precision highp int;
      
        uniform sampler2D texture;
        uniform vec2 delta;
        varying vec2 coord;
      
      
        void main() {
          /* get vertex info */
          vec4 info = texture2D(texture, coord);
      
          /* update the normal */
          vec3 dx = vec3(delta.x, texture2D(texture, vec2(coord.x + delta.x, coord.y)).r - info.r, 0.0);
          vec3 dy = vec3(0.0, texture2D(texture, vec2(coord.x, coord.y + delta.y)).r - info.r, delta.y);
          info.ba = normalize(cross(dy, dx)).xz;
      
          gl_FragColor = info;
        }`
      ;
    };
    
    let simulation_update_fragment;
    function updateSimulationUpdateFragment() {
      simulation_update_fragment = `
        precision highp float;
        precision highp int;
      
        uniform sampler2D texture;
        uniform vec2 delta;
        varying vec2 coord;
      
      
        void main() {
          /* get vertex info */
          vec4 info = texture2D(texture, coord);
      
          /* calculate average neighbor height */
          vec2 dx = vec2(delta.x, 0.0);
          vec2 dy = vec2(0.0, delta.y);
          float average = (
            texture2D(texture, coord - dx).r +
            texture2D(texture, coord - dy).r +
            texture2D(texture, coord + dx).r +
            texture2D(texture, coord + dy).r
          ) * 0.25;
      
          /* change the velocity to move toward the average */
          if (coord.y > 0.495 && coord.y < 0.505 && 
              ${(slit_mode == '0') ? 
                  "((coord.x > 0.0 && coord.x < 0.35) || (coord.x > 0.4 && coord.x < 0.6) || (coord.x > 0.65 && coord.x < 1.0))" 
                :
                  "((coord.x > 0.0 && coord.x < 0.35) || (coord.x > 0.4 && coord.x < 1.0))"
              }
            ) {
            info.g += 0.0;
          }
          else {
            info.g += (average - info.r) * 2.0;
          }
      
          /* attenuate the velocity a little so waves do not last forever */
          info.g *= 0.995;
      
          /* move the vertex along the velocity */
          info.r += info.g;
      
          gl_FragColor = info;
        }`
      ;
    };
    
    let simulation_vertex;
    function updateSimulationVertex() {
      simulation_vertex = `
        attribute vec3 position;
        varying vec2 coord;
      
      
        void main() {
          coord = position.xy * 0.5 + 0.5;
      
          gl_Position = vec4(position.xyz, 1.0);
        }`
      ;
    };
    
    let water_fragment;
    function updateWaterFragment() {
      water_fragment = `
      precision highp float;
      precision highp int;
    
      ${shaders_utils}
    
      uniform float underwater;
      uniform samplerCube sky;
    
      varying vec3 eye;
      varying vec3 pos;
    
    
      vec3 getSurfaceRayColor(vec3 origin, vec3 ray, vec3 waterColor) {
      vec3 color;
  
      ${remove_water ? "return getWallColor(hit);" : `
        if (ray.y < 0.0) {
          vec2 t = intersectSlits(origin, ray, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));
          color = getWallColor(origin + ray * t.y);
        } else {
          vec2 t = intersectSlits(origin, ray, vec3(-1.0, -poolHeight, -1.0), vec3(1.0, 2.0, 1.0));
          vec3 hit = origin + ray * t.y;
          if (hit.y < 7.0 / 12.0) {
            color = getWallColor(hit);
          } else {
            color = textureCube(sky, ray).rgb;
            color += 0.01 * vec3(pow(max(0.0, dot(light, ray)), 20.0)) * vec3(10.0, 8.0, 6.0);
          }
        }
  
        if (ray.y < 0.0) color *= waterColor;
  
        return color;
        `
      }
    }
    
    
      void main() {
        vec2 coord = pos.xz * 0.5 + 0.5;
        vec4 info = texture2D(water, coord);
    
        /* make water look more "peaked" */
        for (int i = 0; i < 5; i++) {
          coord += info.ba * 0.005;
          info = texture2D(water, coord);
        }
    
        vec3 normal = vec3(info.b, sqrt(1.0 - dot(info.ba, info.ba)), info.a);
        vec3 incomingRay = normalize(pos - eye);
    
        if (underwater == 1.) {
          normal = -normal;
          vec3 reflectedRay = reflect(incomingRay, normal);
          vec3 refractedRay = refract(incomingRay, normal, IOR_WATER / IOR_AIR);
          float fresnel = mix(0.5, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));
    
          vec3 reflectedColor = getSurfaceRayColor(pos, reflectedRay, underwaterColor);
          vec3 refractedColor = getSurfaceRayColor(pos, refractedRay, vec3(1.0)) * vec3(0.8, 1.0, 1.1);
    
          gl_FragColor = vec4(mix(reflectedColor, refractedColor, (1.0 - fresnel) * length(refractedRay)), 1.0);
        } else {
          vec3 reflectedRay = reflect(incomingRay, normal);
          vec3 refractedRay = refract(incomingRay, normal, IOR_AIR / IOR_WATER);
          float fresnel = mix(0.25, 1.0, pow(1.0 - dot(normal, -incomingRay), 3.0));
    
          vec3 reflectedColor = getSurfaceRayColor(pos, reflectedRay, abovewaterColor);
          vec3 refractedColor = getSurfaceRayColor(pos, refractedRay, abovewaterColor);
    
          gl_FragColor = vec4(mix(refractedColor, reflectedColor, fresnel), 1.0);
        }
      }`
      ;
    };
    
    let water_vertex;
    function updateWaterVertex() {
      water_vertex = `
        uniform mat4 projectionMatrix;
        uniform mat4 modelViewMatrix;
        uniform sampler2D water;
      
        attribute vec3 position;
      
        varying vec3 eye;
        varying vec3 pos;
      
      
        void main() {
          vec4 info = texture2D(water, position.xy * 0.5 + 0.5);
          pos = position.xzy;
          pos.y += info.r;
      
          vec3 axis_x = vec3(modelViewMatrix[0].x, modelViewMatrix[0].y, modelViewMatrix[0].z);
          vec3 axis_y = vec3(modelViewMatrix[1].x, modelViewMatrix[1].y, modelViewMatrix[1].z);
          vec3 axis_z = vec3(modelViewMatrix[2].x, modelViewMatrix[2].y, modelViewMatrix[2].z);
          vec3 offset = vec3(modelViewMatrix[3].x, modelViewMatrix[3].y, modelViewMatrix[3].z);
      
          eye = vec3(dot(-offset, axis_x), dot(-offset, axis_y), dot(-offset, axis_z));
      
          gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }`
      ;
    };

    function photonReachingWallSampling(threshold=0.5) {
      function P2(x) {
        const beta = 5*Math.PI*x/Math.sqrt(1+x**2);
        const prob = Math.cos(beta)**2 - 0.6*Math.abs(x);
        if (prob >= threshold) {
          return prob;
        }
        else {
          return 0;
        }
      }

      function P1(x) {
        const beta = 5*Math.PI*x/Math.sqrt(1+x**2);
        const prob = (Math.sin(beta)/beta)**2;
        if (prob >= threshold) {
          return prob;
        }
        else {
          return 0;
        }
      }

      function randomUniform(a=-1, b=1) {
        return Math.random() * (b - a) + a;
      }

      function sampleFromP(N=1) {
          const samples = [];
          const maxPDFValue = 1; // This should be an upper bound of P(x). You might need to adjust this.

          while (samples.length < N) {
              const x = randomUniform(); // Choose an appropriate range for x
              const u = randomUniform(0, maxPDFValue);
              if (slit_mode == '0') {
                if (u < P2(x)) {
                    samples.push(x);
                }
              }
              else {
                if (u < P1(x)) {
                    samples.push(x);
                }
              }
          }
          return samples;
      }

      let sample = sampleFromP()[0];
      return sample;
    }
  
    function updateAllshaders() {
      updateShaderUtils();
      updateCausticsFragment();
      updateCausticsVertex();
      updatePoolFragment();
      updatePoolVertex();
      updateSimulationDropFragment();
      updateSimulationNormalFragment();
      updateSimulationUpdateFragment();
      updateSimulationVertex();
      updateWaterFragment();
      updateWaterVertex();
    }
  
    onDestroy( () => {
      if (renderer != undefined) {
        renderer.dispose();
        renderer.forceContextLoss();
      }
    });
  
    afterUpdate(async () => {
      if (document.getElementById("renderer")) {
        document.getElementById("renderer").remove();
      }
  
      if (renderer != undefined) {
        renderer.dispose();
        renderer.forceContextLoss();
      }

      let curPhotonNumber = 0;
      prob_pattern_intensity_front = 0.5;
      prob_pattern_intensity_rear = 0.5;
    
      if (brightness_increase) {
        prob_pattern_intensity_front = 0;
        prob_pattern_intensity_rear = 0;
      }
  
      updateAllshaders();
  
      let width = window.innerWidth;
      let height = window.innerHeight;
      
      // Shader chunks
      THREE.ShaderChunk['utils'] = shaders_utils;
      
      // Create Renderer
      const camera = new THREE.PerspectiveCamera(75, width / height, 0.01, 10000000);
      camera.position.set(0.426, 0.677, -2.095);
      camera.rotation.set(2.828, 0.191, 3.108);
  
      if (renderer != undefined) {
        renderer.dispose();
        renderer.forceContextLoss();
      }
      renderer = new THREE.WebGL1Renderer({antialias: true, alpha: true});
      renderer.setSize(width, height);
      renderer.autoClear = false;
      renderer.domElement.id = "renderer";
      document.getElementById("water-pool-container").appendChild(renderer.domElement);
      
      const spaceModel = await loadModel(spaceUrl, "backgroundSpace");
      spaceModel.translateX(-1.44*1000);
      spaceModel.translateY(-1.44*1000);
      spaceModel.translateZ(1.44*1000);
      spaceModel.scale.set(1000, 1000, 1000);
  
      const backgroundNebulaTexNY = textureloader.load(backgroundNebulaNY);
      const backgroundNebulaTexPY = textureloader.load(backgroundNebulaPY);
      const boxMaterial = [
        new THREE.MeshBasicMaterial({map: textureloader.load(backgroundNebulaNX), side: THREE.BackSide}),
        new THREE.MeshBasicMaterial({map: textureloader.load(backgroundNebulaPX), side: THREE.BackSide}),
        new THREE.MeshBasicMaterial({map: backgroundNebulaTexPY, side: THREE.BackSide}),
        new THREE.MeshBasicMaterial({map: backgroundNebulaTexNY, side: THREE.BackSide}),
        new THREE.MeshBasicMaterial({map: textureloader.load(backgroundNebulaNZ), side: THREE.BackSide}),
        new THREE.MeshBasicMaterial({map: textureloader.load(backgroundNebulaPZ), side: THREE.BackSide}),
      ]
      const backGroundCube = new THREE.Mesh(new THREE.BoxGeometry(1000000, 1000000, 1000000), boxMaterial);
  
      // Light direction
      const light = [0.001, 1.17, 0.001];
      
      // Create mouse Controls
      const controls = new OrbitControls(
        camera,
        renderer.domElement
      );
      
      controls.rotateSpeed = 2.5;
      controls.zoomSpeed = 1.2;
      controls.panSpeed = 0.9;
      controls.dynamicDampingFactor = 0.9;
      
      // Ray caster
      const raycaster = new THREE.Raycaster();
      const mouse = new THREE.Vector2();
      const targetgeometry = new THREE.PlaneGeometry(2, 2);
      targetgeometry.rotateX(-Math.PI / 2);
      const targetmesh = new THREE.Mesh(targetgeometry);
      
      const textureCube = cubetextureloader.load([
        xpos, xneg,
        ypos, ypos,
        zpos, zneg,
      ]);
      
      const tilesTexture = textureloader.load(tiles);
      
      class WaterSimulation {
      
        constructor() {
          this._camera = new THREE.OrthographicCamera(0, 1, 1, 0, 0, 2000);
      
          this._geometry = new THREE.PlaneGeometry(2, 2);
      
          this._textureA = new THREE.WebGLRenderTarget(256, 256, {type: THREE.FloatType});
          this._textureB = new THREE.WebGLRenderTarget(256, 256, {type: THREE.FloatType});
          this.texture = this._textureA;
  
          this.dropMaterial;
          this.normalMaterial;
          this.updateMaterial;
  
          this.init_material = () => {
            this.dropMaterial = new THREE.RawShaderMaterial({
              uniforms: {
                  center: { value: [0, 0] },
                  radius: { value: 0 },
                  strength: { value: 0 },
                  texture: { value: null },
              },
              vertexShader: simulation_vertex,
              fragmentShader: simulation_drop_fragment,
            });
        
            this.normalMaterial = new THREE.RawShaderMaterial({
              uniforms: {
                  delta: { value: [1 / 512, 1 / 512] },  // TODO: Remove this useless uniform and hardcode it in shaders?
                  texture: { value: null },
              },
              vertexShader: simulation_vertex,
              fragmentShader: simulation_normal_fragment,
            });
        
            this.updateMaterial = new THREE.RawShaderMaterial({
              uniforms: {
                  delta: { value: [1 / 512, 1 / 512] },  // TODO: Remove this useless uniform and hardcode it in shaders?
                  texture: { value: null },
              },
              vertexShader: simulation_vertex,
              fragmentShader: simulation_update_fragment,
            });
          };
  
          this.init_material();
  
          this._dropMesh = new THREE.Mesh(this._geometry, this.dropMaterial);
          this._normalMesh = new THREE.Mesh(this._geometry, this.normalMaterial);
          this._updateMesh = new THREE.Mesh(this._geometry, this.updateMaterial);
  
          this.update_material = () => {
            this.init_material();
  
            this._dropMesh.material = this.dropMaterial;
            this._normalMesh.material = this.normalMaterial;
            this._updateMesh.material = this.updateMaterial;
  
            this._dropMesh.material.needsUpdate = true;
            this._normalMesh.material.needsUpdate = true;
            this._updateMesh.material.needsUpdate = true;
          }
        }
      
        // Add a drop of water at the (x, y) coordinate (in the range [-1, 1])
        addDrop(renderer, x, y, radius, strength) {
          this._dropMesh.material.uniforms['center'].value = [x, y];
          this._dropMesh.material.uniforms['radius'].value = radius;
          this._dropMesh.material.uniforms['strength'].value = strength;
      
          this._render(renderer, this._dropMesh);
        }
      
        stepSimulation(renderer) {
          this._render(renderer, this._updateMesh);
        }
      
        updateNormals(renderer) {
          this._render(renderer, this._normalMesh);
        }
      
        _render(renderer, mesh) {
          // Swap textures
          const oldTexture = this.texture;
          const newTexture = this.texture === this._textureA ? this._textureB : this._textureA;
      
          mesh.material.uniforms['texture'].value = oldTexture.texture;
      
          renderer.setRenderTarget(newTexture);
      
          // TODO Camera is useless here, what should be done?
          renderer.render(mesh, this._camera);
      
          this.texture = newTexture;
        }
      
      }
      
      
      class Caustics {
      
        constructor(lightFrontGeometry) {
          this._camera = new THREE.OrthographicCamera(0, 1, 1, 0, 0, 2000);
      
          this._geometry = lightFrontGeometry;
      
          this.texture = new THREE.WebGLRenderTarget(1024, 1024, {type: THREE.UNSIGNED_BYTE});
      
          this.material;
  
          this.init_material = () => {
            this.material = new THREE.RawShaderMaterial({
              uniforms: {
                  light: { value: light },
                  water: { value: null },
              },
              vertexShader: caustics_vertex,
              fragmentShader: caustics_fragment,
            });
          };
  
          this.init_material();
  
          this._causticMesh = new THREE.Mesh(this._geometry, this.material);
  
          this.update_material = () => {
            this.init_material();
  
            this._causticMesh.material = this.material;
  
            this._causticMesh.material.needsUpdate = true;
          }
        }
      
        update(renderer, waterTexture) {
          this._causticMesh.material.uniforms['water'].value = waterTexture;
      
          renderer.setRenderTarget(this.texture);
          renderer.clear();
      
          // TODO Camera is useless here, what should be done?
          renderer.render(this._causticMesh, this._camera);
        }
      
      }
      
      
      class Water {
      
        constructor() {
          this.geometry = new THREE.PlaneGeometry(2, 2, 200, 200);
      
          this.material;
  
          this.init_material = () => {
            this.material = new THREE.RawShaderMaterial({
              uniforms: {
                  light: { value: light },
                  tiles: { value: tilesTexture },
                  sky: { value: textureCube },
                  water: { value: null },
                  causticTex: { value: null },
                  underwater: { value: false },
              },
              vertexShader: water_vertex,
              fragmentShader: water_fragment,
            });
          }
  
          this.init_material();
  
          this._mesh = new THREE.Mesh(this.geometry, this.material);
  
          this.update_material = () => {
            this.init_material();
  
            this._mesh.material = this.material;
            
            this._mesh.material.needsUpdate = true;
          }
        }
      
        draw(renderer, waterTexture, causticsTexture) {
          this.material.uniforms['water'].value = waterTexture;
          this.material.uniforms['causticTex'].value = causticsTexture;
      
          this.material.side = THREE.FrontSide;
          this.material.uniforms['underwater'].value = true;
          renderer.render(this._mesh, camera);
      
          this.material.side = THREE.BackSide;
          this.material.uniforms['underwater'].value = false;
          renderer.render(this._mesh, camera);
        }
      
      }
      
      
      class Pool {
      
        constructor() {
          this._geometry = new THREE.BufferGeometry();
          const vertices = new Float32Array([
            -1, -1, -1,
            -1, -1, 1,
            -1, 1, -1,
            -1, 1, 1,
            1, -1, -1,
            1, 1, -1,
            1, -1, 1,
            1, 1, 1,
            -1, -1, -1,
            1, -1, -1,
            -1, -1, 1,
            1, -1, 1,
            -1, 1, -1,
            -1, 1, 1,
            1, 1, -1,
            1, 1, 1,
            -1, -1, -1,
            -1, 1, -1,
            1, -1, -1,
            1, 1, -1,
            -1, -1, 1,
            1, -1, 1,
            -1, 1, 1,
            1, 1, 1,
          ]);
          const indices = new Uint32Array([
            0, 1, 2,
            2, 1, 3,
            4, 5, 6,
            6, 5, 7,
            12, 13, 14,
            14, 13, 15,
            16, 17, 18,
            18, 17, 19,
            20, 21, 22,
            22, 21, 23,
          ]);
      
          this._geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
          this._geometry.setIndex(new THREE.BufferAttribute(indices, 1));
      
          this.material;
  
          this.init_material = () => {
            this.material = new THREE.RawShaderMaterial({
              uniforms: {
                  light: { value: light },
                  tiles: { value: tilesTexture },
                  water: { value: null },
                  causticTex: { value: null },
              },
              vertexShader: pool_vertex,
              fragmentShader: pool_fragment,
            });
            this.material.side = THREE.FrontSide;
          };
          this.init_material();
  
          this._mesh = new THREE.Mesh(this._geometry, this.material);
  
          this.update_material = () => {
            this.init_material();
  
            this._mesh.material = this.material;
            
            this._mesh.material.needsUpdate = true;
          }
        }
      
        draw(renderer, waterTexture, causticsTexture) {
          this.material.uniforms['water'].value = waterTexture;
          this.material.uniforms['causticTex'].value = causticsTexture;
  
          renderer.render(this._mesh, camera);
        }
      
      }
      
      class Slits {
        constructor() {
          this._geometry = new THREE.BufferGeometry();
          let vertices;
          if (slit_mode == '0' || slit_mode == '2') {
            vertices = new Float32Array([
              -1, -1, 0,
              -0.3, -1, 0,
              -0.3, 1, 0,
            
              -0.3, 1, 0,
              -1,  1, 0,
              -1, -1, 0,
      
              -0.2, -1, 0,
              0.2, -1, 0,
              0.2,  1, 0,
      
              0.2,  1, 0,
              -0.2,  1, 0,
              -0.2, -1, 0,
      
              0.3, -1, 0,
              1, -1, 0,
              1,  1, 0,
      
              1,  1, 0,
              0.3, 1, 0,
              0.3, -1, 0,
            ]);
          }
          if (slit_mode == '1') {
            vertices = new Float32Array([
              -1, -1, 0,
              -0.3, -1, 0,
              -0.3, 1, 0,
            
              -0.3, 1, 0,
              -1,  1, 0,
              -1, -1, 0,
      
              -0.2, -1, 0,
              0.2, -1, 0,
              0.2,  1, 0,
      
              0.2,  1, 0,
              -0.2,  1, 0,
              -0.2, -1, 0,
      
              0.2, -1, 0,
              1, -1, 0,
              1,  1, 0,
      
              1,  1, 0,
              0.2, 1, 0,
              0.2, -1, 0,
            ]);
          }
          const indices = new Uint32Array([
            0, 1, 2,
            3, 4, 5,
            6, 7, 8,
            9, 10, 11,
            12, 13, 14,
            15, 16, 17,
          ]);
  
          if (slit_mode == '2') {
            this._geometry2 = new THREE.ConeGeometry(0.05, 0.1, 8, 8);
            this.material2 = new THREE.MeshBasicMaterial({color: 0x00f0f0});
            this._mesh2 = new THREE.Mesh(this._geometry2, this.material2);
            this._mesh2.position.set(-0.25, 0.116, 0);
            this._mesh2.rotateX(Math.PI);
          }
      
          this._geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
          this._geometry.setIndex(new THREE.BufferAttribute(indices, 1));
      
      
          this.material;
          this.init_material = () => {
            this.material = new THREE.RawShaderMaterial({
              uniforms: {
                  light: { value: light },
                  tiles: { value: tilesTexture },
                  water: { value: null },
                  causticTex: { value: null },
              },
              vertexShader: pool_vertex,
              fragmentShader: pool_fragment,
            });
            this.material.side = THREE.DoubleSide;
          };
          this.init_material();
  
          this._mesh = new THREE.Mesh(this._geometry, this.material);
  
          this.update_material = () => {
            this.init_material();
  
            this._mesh.material = this.material;
            
            this._mesh.material.needsUpdate = true;
          }
        }
      
        draw(renderer, waterTexture, causticsTexture)  {
          this.material.uniforms['water'].value = waterTexture;
          this.material.uniforms['causticTex'].value = causticsTexture;
      
          renderer.render(this._mesh, camera);
          if (this._mesh2) {
            renderer.render(this._mesh2, camera);
          }
        }
      }
  
      class Photon {
        constructor() {
          this._geometry = new THREE.SphereGeometry(0.02, 8, 8);
          this._material = new THREE.MeshBasicMaterial({color: 0xfff700, transparent: true});
          this._mesh1 = new THREE.Mesh(this._geometry, this._material);
          this._mesh2 = new THREE.Mesh(this._geometry, this._material);
          this._wave_speed = 1/256;
          this.initial_position;
          this._vector1;
          this._vector2;
          this.position1;
          this.position2;
          this.reachedEndTime;
        }
  
        place(x, y, z) {
          this.position1 = {x, y, z};
          this.position2 = {x, y, z};
          this.initial_position = {x, y, z};
  
          this._vector1 = {x: -0.25 - x , y: -y, z: -z};
          this._vector2 = {x: 0.25 - x , y: -y, z: -z};
          const magnitude1 = Math.sqrt(this._vector1.x**2 + this._vector1.y**2 + this._vector1.z**2);
          const magnitude2 = Math.sqrt(this._vector2.x**2 + this._vector2.y**2 + this._vector2.z**2);
          this._vector1 = {x: this._vector1.x/magnitude1, y: this._vector1.y/magnitude1, z: this._vector1.z/magnitude1};
          this._vector2 = {x: this._vector2.x/magnitude2, y: this._vector2.y/magnitude2, z: this._vector2.z/magnitude2};
  
          this._mesh1.position.set(this.position1.x, this.position1.y, this.position1.z);
          this._mesh2.position.set(this.position2.x, this.position2.y, this.position2.z);
        }
      
        draw(renderer, waterSimulation_update_material, water_update_material, caustics_update_material, pool_update_material, slits_update_material)  {
          const addWallBrightness = () => {
            if (this.initial_position.z > 0) {
              prob_pattern_intensity_front += 0.1;
            }
            else {
              prob_pattern_intensity_rear += 0.1;
            }
            updateAllshaders();
            waterSimulation_update_material();
            water_update_material();
            caustics_update_material();
            pool_update_material();
            slits_update_material();
          }
  
          if (this.initial_position && !this.reachedEndTime) {
            if (!pause) {
              this.position1 = {
                x: this.position1.x + this._vector1.x * this._wave_speed,
                y: this.position1.y + this._vector1.y * this._wave_speed,
                z: this.position1.z + this._vector1.z * this._wave_speed
              };
              this.position2 = {
                x: this.position2.x + this._vector2.x * this._wave_speed,
                y: this.position2.y + this._vector2.y * this._wave_speed,
                z: this.position2.z + this._vector2.z * this._wave_speed
              };
            }
  
            if (Math.abs(this.position1.z - this.initial_position.z) < 0.05) {
              this._mesh1.position.set((this.position1.x + this.position2.x)/2, this.position1.y, this.position1.z);
              renderer.render(this._mesh1, camera);
            }
  
            if (slit_mode == '0') {
              if (Math.abs(this.position1.z) < 0.1) {
                this._mesh1.material.opacity = 0.4;
                this._mesh1.position.set(this.position1.x, this.position1.y, this.position1.z);
                renderer.render(this._mesh1, camera);
              }
              if (Math.abs(this.position2.z) < 0.1) {
                this._mesh2.material.opacity = 0.4;
                this._mesh2.position.set(this.position2.x, this.position2.y, this.position2.z);
                renderer.render(this._mesh2, camera);
              }
  
              if (((this.position1.z > 0.95 || this.position2.z > 0.95) && this.initial_position.z < 0) || ((this.position1.z < -0.95 || this.position2.z < -0.95) && this.initial_position.z >= 0)) {
                if (this.initial_position.z < 0) {
                  this._mesh1.position.set(photonReachingWallSampling(), 0, 0.99);
                }
                else {
                  this._mesh1.position.set(photonReachingWallSampling(), 0, -0.99);
                }
                this.reachedEndTime = new Date();
                this._mesh2.material.opacity = 1;
                renderer.render(this._mesh1, camera);
                if (brightness_increase) {
                  addWallBrightness();
                }
              }
            }
            else {
              if (Math.abs(this.position1.z) < 0.1) {
                this._mesh1.position.set(this.position1.x, this.position1.y, this.position1.z);
                renderer.render(this._mesh1, camera);
              }
  
              if (((this.position1.z > 0.95 || this.position2.z > 0.95) && this.initial_position.z < 0) || ((this.position1.z < -0.95 || this.position2.z < -0.95) && this.initial_position.z >= 0)) {
                if (this.initial_position.z < 0) {
                  this._mesh1.position.set(photonReachingWallSampling()-0.25, 0, 0.99);
                }
                else {
                  this._mesh1.position.set(photonReachingWallSampling()-0.25, 0, -0.99);
                }
                this.reachedEndTime = new Date();
                this._mesh2.material.opacity = 1;
                renderer.render(this._mesh1, camera);
                if (brightness_increase) {
                  addWallBrightness();
                }
              }
            }
          }
          if (this.reachedEndTime) {
            if (pause) {
              this.reachedEndTime = new Date();
            }
            renderer.render(this._mesh1, camera);
          }
        }
      }
  
      class Photons {
        constructor() {
          this._photons = [];
        }
  
        place(x, y, z) {
          this._photons.push(new Photon());
          this._photons[this._photons.length - 1].place(x, y, z);
          curPhotonNumber++;
        }
      
        draw(renderer, waterSimulation_update_material, water_update_material, caustics_update_material, pool_update_material, slits_update_material) {
          const curTime = new Date();
          for (let photon of this._photons) {
            if (photon.reachedEndTime && curTime - photon.reachedEndTime > 3000) {
              if (!photon["decremented"]) {
                curPhotonNumber--;
                photon["decremented"] = true;
              }
              if (remove_photons) {
                let index = this._photons.findIndex(obj => obj === photon);
    
                // Remove the object if it exists
                if (index !== -1) {
                  this._photons.splice(index, 1);
                }
              }
            }
            photon.draw(renderer, waterSimulation_update_material, water_update_material, caustics_update_material, pool_update_material, slits_update_material);
          }
        }
      }
      
      const waterSimulation = new WaterSimulation();
      const water = new Water();
      const caustics = new Caustics(water.geometry);
      const pool = new Pool();
      const slits = new Slits();
      const photons = new Photons();
      
      
      // Main rendering loop
      function animate() {
        if (!pause) {
          waterSimulation.stepSimulation(renderer);
          waterSimulation.updateNormals(renderer);
        }
        
        const waterTexture = waterSimulation.texture.texture;
        
        caustics.update(renderer, waterTexture);
        
        const causticsTexture = caustics.texture.texture;
      
        renderer.setRenderTarget(null);
        renderer.clear();
    
        pool.draw(renderer, waterTexture, causticsTexture);
        water.draw(renderer, waterTexture, causticsTexture);
        slits.draw(renderer, waterTexture, causticsTexture);
        photons.draw(renderer, waterSimulation.update_material, water.update_material, caustics.update_material, pool.update_material, slits.update_material);
        renderer.render(spaceModel, camera);
        renderer.render(backGroundCube, camera);
  
        controls.update();
      
        window.requestAnimationFrame(animate);
      }
      
      function onMouseDown(event) {
        const rect = renderer.domElement.getBoundingClientRect();
      
        mouse.x = (event.clientX - rect.left) * 2 / width - 1;
        mouse.y = - (event.clientY - rect.top) * 2 / height + 1;
      
        raycaster.setFromCamera(mouse, camera);
      
        const intersects = raycaster.intersectObject(targetmesh);
      
        for (let intersect of intersects) {
          waterSimulation.addDrop(renderer, intersect.point.x, intersect.point.z, 0.03, 0.04);
          if (place_photon && (allow_many || curPhotonNumber < 1)) {
            photons.place(intersect.point.x, 0.02, intersect.point.z);
          }
        }
      }
      
      renderer.domElement.addEventListener('mousedown', { handleEvent: onMouseDown });
  
      window.addEventListener('resize', () => {
        width = window.innerWidth;
        height = window.innerHeight;
        camera.aspect = width/height;
        camera.updateProjectionMatrix();
        renderer.setSize(width, height);
      });
      
      animate();
    });
</script>
  
<div id="water-pool-container">

</div>