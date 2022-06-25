<template>
  <div id="container">
    <div class="popup" v-if="gameOver">
      <span>gg啦</span>
      <button type="button" @click="restart">重新开始</button>
    </div>
  </div>
</template>

<script>
import * as THREE from "three";
import { defineComponent, watch, onMounted, ref } from "vue";
// 调色盘
const Colors = {
  red: 0xf25346,
  white: 0xd8d0d1,
  brown: 0x59332e,
  pink: 0xf5986e,
  brownDark: 0x23190f,
  blue: 0x68c3c0,
};

export default defineComponent({
  name: "myGame",
  setup(props) {
    let scene,
      camera,
      fieldOfView,
      aspectRatio,
      nearPlane,
      farPlane,
      HEIGHT,
      WIDTH,
      renderer,
      container,
      hemisphereLight,
      shadowLight,
      airplane,
      sky,
      bulletsArray = [],
      enemyArray = [],
      // bullets,
      sea;

    let mousePos = { x: 0, y: 0 };
    let gameOver = ref(false)
    function randomIntFromInterval(min, max) { // min and max included 
      return Math.floor(Math.random() * (max - min + 1) + min)
    }

    const createScene = () => {
      // Get the width and the height of the screen,
      // use them to set up the aspect ratio of the camera
      // and the size of the renderer.
      HEIGHT = window.innerHeight;
      WIDTH = window.innerWidth;

      // Create the scene
      scene = new THREE.Scene();

      // Add a fog effect to the scene; same color as the
      // background color used in the style sheet
      scene.fog = new THREE.Fog(0xf7d9aa, 100, 950);

      // Create the camera
      aspectRatio = WIDTH / HEIGHT;
      fieldOfView = 60;
      nearPlane = 1;
      farPlane = 10000;
      camera = new THREE.PerspectiveCamera(
        fieldOfView,
        aspectRatio,
        nearPlane,
        farPlane
      );

      // Set the position of the camera
      camera.position.x = 0;
      camera.position.z = 200;
      camera.position.y = 100;

      // Create the renderer
      renderer = new THREE.WebGLRenderer({
        // Allow transparency to show the gradient background
        // we defined in the CSS
        alpha: true,

        // Activate the anti-aliasing; this is less performant,
        // but, as our project is low-poly based, it should be fine :)
        antialias: true,
      });

      // Define the size of the renderer; in this case,
      // it will fill the entire screen
      renderer.setSize(WIDTH, HEIGHT);

      // Enable shadow rendering
      renderer.shadowMap.enabled = true;

      // Add the DOM element of the renderer to the
      // container we created in the HTML
      container = document.getElementById("container");
      container.appendChild(renderer.domElement);

      // Listen to the screen: if the user resizes it
      // we have to update the camera and the renderer size
      // window.addEventListener("resize", handleWindowResize, false);
    };

    const createLights = () => {
      hemisphereLight = new THREE.HemisphereLight(0xaaaaaa, 0x000000, 0.9);

      // A directional light shines from a specific direction.
      // It acts like the sun, that means that all the rays produced are parallel.
      shadowLight = new THREE.DirectionalLight(0xffffff, 0.9);

      // Set the direction of the light
      shadowLight.position.set(150, 350, 350);

      // Allow shadow casting
      shadowLight.castShadow = true;

      // define the visible area of the projected shadow
      shadowLight.shadow.camera.left = -400;
      shadowLight.shadow.camera.right = 400;
      shadowLight.shadow.camera.top = 400;
      shadowLight.shadow.camera.bottom = -400;
      shadowLight.shadow.camera.near = 1;
      shadowLight.shadow.camera.far = 1000;

      // define the resolution of the shadow; the higher the better,
      // but also the more expensive and less performant
      shadowLight.shadow.mapSize.width = 2048;
      shadowLight.shadow.mapSize.height = 2048;

      // to activate the lights, just add them to the scene
      scene.add(hemisphereLight);
      scene.add(shadowLight);
    };

    const createSea = () => {
      function Sea() {
        // create the geometry (shape) of the cylinder;
        // the parameters are:
        // radius top, radius bottom, height, number of segments on the radius, number of segments vertically
        var geom = new THREE.CylinderGeometry(600, 600, 800, 20, 10, true);

        // rotate the geometry on the x axis
        geom.applyMatrix4(new THREE.Matrix4().makeRotationX(-Math.PI / 2));

        // create the material
        var mat = new THREE.MeshPhongMaterial({
          color: '#53c726',
          transparent: true,
          opacity: 0.6,
          shading: THREE.FlatShading,
        });

        // To create an object in Three.js, we have to create a mesh
        // which is a combination of a geometry and some material
        this.mesh = new THREE.Mesh(geom, mat);

        // Allow the sea to receive shadows
        this.mesh.receiveShadow = true;
      }

      sea = new Sea();

      // push it a little bit at the bottom of the scene
      sea.mesh.position.y = -600;

      // add the mesh of the sea to the scene
      scene.add(sea.mesh);
    };

    const createPlane = () => {
      function AirPlane() {
        this.mesh = new THREE.Object3D();

        // Create the cabin
       const geometry = new THREE.CapsuleGeometry( 20, 70, 4, 8 );
        const matCockpit = new THREE.MeshPhongMaterial({
          color: Colors.red,
          shading: THREE.FlatShading,
        });
        const cockpit = new THREE.Mesh( geometry, matCockpit );
        cockpit.rotation.z= Math.PI / 2;
        cockpit.castShadow = true;
        cockpit.receiveShadow = true;
        this.mesh.add(cockpit);

      // Create the room
       const geometryRoom = new THREE.CapsuleGeometry( 20, 20, 4, 8 );
        const roomMaterial = new THREE.MeshPhongMaterial({
          color: Colors.brown,
          shading: THREE.FlatShading,
        });
        const room = new THREE.Mesh( geometryRoom, roomMaterial );
        room.position.set(15, 15, 0);

        room.rotation.z= Math.PI / 2;
        this.mesh.add(room);


        // Create the tail
        var geomTailPlane = new THREE.BoxGeometry(15, 20, 5, 1, 1, 1);
        var matTailPlane = new THREE.MeshPhongMaterial({
          color: Colors.red,
          shading: THREE.FlatShading,
        });
        var tailPlane = new THREE.Mesh(geomTailPlane, matTailPlane);
        tailPlane.position.set(-48, 20, 0);
        tailPlane.castShadow = true;
        tailPlane.receiveShadow = true;
        this.mesh.add(tailPlane);

        // Create the wing
        var geomSideWing = new THREE.BoxGeometry(40, 8, 200, 1, 1, 1);
        var matSideWing = new THREE.MeshPhongMaterial({
          color: Colors.white,
          shading: THREE.FlatShading,
        });
        var sideWing = new THREE.Mesh(geomSideWing, matSideWing);
        sideWing.castShadow = true;
        sideWing.receiveShadow = true;
        this.mesh.add(sideWing);

        // propeller
        var geomPropeller = new THREE.BoxGeometry(20, 10, 10, 1, 1, 1);
        var matPropeller = new THREE.MeshPhongMaterial({
          color: Colors.brown,
          shading: THREE.FlatShading,
        });
        this.propeller = new THREE.Mesh(geomPropeller, matPropeller);
        this.propeller.castShadow = true;
        this.propeller.receiveShadow = true;

        // blades
        var geomBlade = new THREE.BoxGeometry(1, 100, 20, 1, 1, 1);
        var matBlade = new THREE.MeshPhongMaterial({
          color: Colors.brownDark,
          shading: THREE.FlatShading,
        });

        var blade = new THREE.Mesh(geomBlade, matBlade);
        blade.position.set(8, 0, 0);
        blade.castShadow = true;
        blade.receiveShadow = true;
        this.propeller.add(blade);
        this.propeller.position.set(50, 0, 0);
        this.mesh.add(this.propeller);
      }
      airplane = new AirPlane();
      airplane.mesh.scale.set(0.25, 0.25, 0.25);
      airplane.mesh.position.y = 100;
      scene.add(airplane.mesh);
    };

    function updatePlane() {
      // let's move the airplane between -100 and 100 on the horizontal axis,
      // and between 25 and 175 on the vertical axis,
      // depending on the mouse position which ranges between -1 and 1 on both axes;
      // to achieve that we use a normalize function (see below)
      
      var targetX = normalize(mousePos.x, -1, 1, -100, 100);
      var targetY = normalize(mousePos.y, -1, 1, 25, 175);

      // update the airplane's position
      airplane.mesh.position.y = targetY;
      airplane.mesh.position.x = targetX;
      airplane.propeller.rotation.x += 0.3;

      // console.log('y', airplane.mesh.position.y)
      // console.log('x', airplane.mesh.position.x)
    }

    function normalize(v, vmin, vmax, tmin, tmax) {
      var nv = Math.max(Math.min(v, vmax), vmin);
      var dv = vmax - vmin;
      var pc = (nv - vmin) / dv;
      var dt = tmax - tmin;
      var tv = tmin + pc * dt;
      return tv;
    }

    function loop() {
      // Rotate the propeller, the sea and the sky
      airplane.propeller.rotation.x += 0.3;
      sea.mesh.rotation.z += 0.005;
      sky.mesh.rotation.z += 0.002;
      updatePlane();

      // render the scene
      renderer.render(scene, camera);
  
      // call the loop function again
      requestAnimationFrame(loop);
    }

    function handleMouseMove(event) {
      if(gameOver.value) {
        return
      }
      // here we are converting the mouse position value received
      // to a normalized value varying between -1 and 1;
      // this is the formula for the horizontal axis:
      var tx = -1 + (event.targetTouches[0].clientX / WIDTH) * 2;

      // for the vertical axis, we need to inverse the formula
      // because the 2D y-axis goes the opposite direction of the 3D y-axis

      var ty = 1 - (event.targetTouches[0].clientY / HEIGHT) * 2;
      mousePos = { x: tx, y: ty };
    }

    function createSky() {
      function Cloud(){
        // Create an empty container that will hold the different parts of the cloud
        this.mesh = new THREE.Object3D();
        
        // create a cube geometry;
        // this shape will be duplicated to create the cloud
        var geom = new THREE.IcosahedronGeometry(20,0);
        
        // create a material; a simple white material will do the trick
        var mat = new THREE.MeshPhongMaterial({
          color: Colors.white,  
        });
        
        // duplicate the geometry a random number of times
        var nBlocs = 3+Math.floor(Math.random()*3);
        for (var i=0; i<nBlocs; i++ ){
          
          // create the mesh by cloning the geometry
          var m = new THREE.Mesh(geom, mat); 
          
          // set the position and the rotation of each cube randomly
          m.position.x = i*15;
          m.position.y = Math.random()*10;
          m.position.z = Math.random()*10;
          m.rotation.z = Math.random()*Math.PI*2;
          m.rotation.y = Math.random()*Math.PI*2;
          
          // set the size of the cube randomly
          var s = .1 + Math.random()*.9;
          m.scale.set(s,s,s);
          
          // allow each cube to cast and to receive shadows
          m.castShadow = true;
          m.receiveShadow = true;
          
          // add the cube to the container we first created
          this.mesh.add(m);
        } 
      }
      function Sky() {
        this.mesh = new THREE.Object3D();
        
        // choose a number of clouds to be scattered in the sky
        this.nClouds = 20;
        
        // To distribute the clouds consistently,
        // we need to place them according to a uniform angle
        var stepAngle = Math.PI*2 / this.nClouds;
        
        // create the clouds
        for(var i=0; i<this.nClouds; i++){
          var c = new Cloud();
        
          // set the rotation and the position of each cloud;
          // for that we use a bit of trigonometry
          var a = stepAngle*i; // this is the final angle of the cloud
          var h = 750 + Math.random()*200; // this is the distance between the center of the axis and the cloud itself

          // Trigonometry!!! I hope you remember what you've learned in Math :)
          // in case you don't: 
          // we are simply converting polar coordinates (angle, distance) into Cartesian coordinates (x, y)
          c.mesh.position.y = Math.sin(a)*h;
          c.mesh.position.x = Math.cos(a)*h;

          // rotate the cloud according to its position
          c.mesh.rotation.z = a + Math.PI/2;

          // for a better result, we position the clouds 
          // at random depths inside of the scene
          c.mesh.position.z = -400-Math.random()*400;
          
          // we also set a random scale for each cloud
          var s = 1+Math.random()*2;
          c.mesh.scale.set(s,s,s);

          // do not forget to add the mesh of each cloud in the scene
          this.mesh.add(c.mesh);  
        }  
      }
      sky = new Sky();
      sky.mesh.position.y = -600;
      scene.add(sky.mesh);
    }

    function createBullet() {
        function BULLET() {
          const geometry = new THREE.CapsuleGeometry( 1, 1, 4, 8 );
          const bulletMaterial = new THREE.MeshPhongMaterial({
            color: Colors.brown,
            shading: THREE.FlatShading,
          });
          const bullet = new THREE.Mesh( geometry, bulletMaterial );
          bullet.rotation.z= Math.PI / 2;
          bullet.position.set(70, 0, 0);
          this.mesh = bullet

          this.mesh.position.x = airplane.mesh.position.x
          this.mesh.position.y =  airplane.mesh.position.y

          this.fire = () => {
              if(this.mesh.position.x >= 100) {
                  const index = bulletsArray.indexOf(this);
                  if (index > -1) {
                    bulletsArray.splice(index, 1);
                  }
                 cancelAnimationFrame(this.animate)
                 this.mesh.clear()
                 return
              }
              this.mesh.position.x += 2;
              requestAnimationFrame(this.fire);
          }
        }

        setInterval(() => {
          if( gameOver.value ) {
            return
          }
          let bullet = new BULLET()
          scene.add(bullet.mesh)
          bulletsArray.push(bullet)

          bullet.fire()
        }, 200)

        
    }

    function createEnemy() {
        function Enemy(){
          this.mesh = new THREE.Object3D();

          const enemyOutline = new THREE.CapsuleGeometry( 2, 10, 40, 8 );
          const enemyMaterial = new THREE.MeshPhongMaterial({
            color: Colors.brown,
            shading: THREE.FlatShading,
          });
          const enemy = new THREE.Mesh( enemyOutline, enemyMaterial);
          enemy.rotation.z= Math.PI / 2;


        var geomSideWing = new THREE.BoxGeometry(5, 1, 20, 1, 1, 1);
        var matSideWing = new THREE.MeshPhongMaterial({
          color: Colors.blue,
          shading: THREE.FlatShading,
        });
        var sideWing = new THREE.Mesh(geomSideWing, matSideWing);
        sideWing.castShadow = true;
        sideWing.receiveShadow = true;
        this.mesh.add(sideWing);

 var geomSideWing2 = new THREE.BoxGeometry(5, 1, 10, 1, 1, 1);
        var matSideWing2 = new THREE.MeshPhongMaterial({
          color: Colors.blue,
          shading: THREE.FlatShading,
        });
        var sideWing2 = new THREE.Mesh(geomSideWing2, matSideWing2);
        sideWing2.castShadow = true;
        sideWing2.receiveShadow = true;
                sideWing2.position.set(7, 0, 0);

        this.mesh.add(sideWing2);


          enemy.castShadow = true;
          enemy.receiveShadow = true;

          this.mesh.add(enemy)
          const y = randomIntFromInterval(20, 200)
          this.mesh.position.y = y
          this.mesh.position.x = 150
          const ySpeed = randomIntFromInterval(-10, 10) / 10;
          this.died = false

          this.checkCollision = () => {
            const top = this.mesh.position.y + 10;
            const bottom = this.mesh.position.y - 10;
            const left = this.mesh.position.x - 10;
            const right = this.mesh.position.x + 10;
            for(let i = 0 ; i < bulletsArray.length; i++) {
              // 检查是否被子弹击中
              if(bulletsArray[i].mesh.position.x > left && bulletsArray[i].mesh.position.x < right) {
                if(bulletsArray[i].mesh.position.y > bottom && bulletsArray[i].mesh.position.y < top) {
                  //  scene.delete(this.mesh)
                  const index = enemyArray.indexOf(this);
                  if (index > -1) {
                    enemyArray.splice(index, 1);
                  }

                  //eslint-disable-next-line
                  this.died = true
                  this.mesh.clear()
                  return;
                }
              }
            }

            if(airplane.mesh.position.x > left && airplane.mesh.position.x <right) {
              if(airplane.mesh.position.y > bottom && airplane.mesh.position.y < top) {
                // alert('Died!')
                gameOver.value = true;
                airplane.mesh.clear()
              }
            }
          }
    
          this.move = () => {
            if(this.died) {
              return
            }
            if(this.mesh.position.x <= -100) {
                  const index = enemyArray.indexOf(this);
                  if (index > -1) {
                    enemyArray.splice(index, 1);
                  }
                this.mesh.clear()
                return
            }
            this.checkCollision()
            this.mesh.position.x -= 1;
            if(this.mesh.position.y >= 20) {
            this.mesh.position.y += ySpeed;
            }
            requestAnimationFrame(this.move);
          }
        }

        setInterval(() => {
          let enemy = new Enemy()
           scene.add(enemy.mesh)
          enemyArray.push(enemy)
           enemy.move()
        }, 100)
    }
    onMounted(async () => {
      createScene();

      // add the lights
      createLights();

      // // add the objects
      createPlane();
      createSea();
      createSky();
      createBullet();
      createEnemy();
      // // start a loop that will update the objects' positions
      // // and render the scene on each frame

      // document.addEventListener("mousemove", handleMouseMove, false);
      document.addEventListener("touchmove", handleMouseMove, false);

      loop();
    });
    watch(props, () => {});
    return {
      restart: () => {
        location.reload()
      },
      gameOver,
      cancelModal: () => {},
      submitForm: async () => {},
    };
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#container {
  touch-action: none;
  position: absolute;
  width: 100%;
  height: 100%;
  overflow: hidden;
  display: flex;
    align-items: center;
    flex-direction: column;
  background: linear-gradient(to bottom, #b7eaff 0%,#94dfff 100%);
}

.popup {
   position: absolute;
    top: 300px;
    height: 100px;
    width: 300px;
    background: black;
    opacity: 0.5;
    display: flex;
    /* flex-direction: column-reverse; */
    align-items: center;
    flex-direction: column;
    /* flex: revert; */
    justify-content: center;
}

  span {
      color: white;
      margin-bottom: 20px;
  }
</style>
