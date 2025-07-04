<template>
  <div id="game-container">
    <div id="crosshair"></div>
    <div id="ui">
      <div>WASD: Move | Mouse: Look | Space: Jump</div>
      <div>O: Toggle Camera | Click to lock cursor</div>
      <div id="debug-info">
        <div>Position: <span id="position">0, 0, 0</span></div>
        <div>Velocity: <span id="velocity">0, 0, 0</span></div>
        <div>Grounded: <span id="grounded">false</span></div>
        <div>In Vehicle: <span id="in-vehicle">false</span></div>
        <div>Camera: <span id="camera-mode">First Person</span></div>
        <div>Intersections: <span id="intersections">0</span></div>
        <div>Sensors: <span id="sensors">0</span></div>
        <div>Vehicle Up: <span id="vehicle-up">0, 1, 0</span></div>
        <div>Vehicle Forward: <span id="vehicle-forward">0, 0, -1</span></div>
      </div>
    </div>
    <canvas ref="canvas"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import RAPIER from '@dimforge/rapier3d-compat'

const canvas = ref(null)

let scene, camera, renderer, world
let player, playerBody, playerController
let vehicle, vehicleBody, interiorSensors = []
let playerCollider = null
let isPlayerInVehicle = false
let debugRayHelper = null
let isThirdPerson = false
let bayDebugMesh = null
let floorDebugMesh = null

class FPSController {
  constructor(camera, body, world) {
    this.camera = camera
    this.body = body
    this.world = world
    this.velocity = new THREE.Vector3()
    this.direction = new THREE.Vector3()
    this.moveSpeed = 5
    this.jumpForce = 6
    this.mouseSensitivity = 0.002
    this.isGrounded = false
    this.keys = {}
    this.euler = new THREE.Euler(0, 0, 0, 'YXZ')
    this.isLocked = false
    this.isInVehicle = false
    this.vehicleTransform = null
    this.groundRayOrigin = new THREE.Vector3()
    this.groundRayDir = new THREE.Vector3(0, -1, 0)
    
    this.setupControls()
  }

  setupControls() {
    document.addEventListener('keydown', (e) => {
      this.keys[e.code] = true
      
      if (e.code === 'KeyO') {
        isThirdPerson = !isThirdPerson
        document.getElementById('camera-mode').textContent = isThirdPerson ? 'Third Person' : 'First Person'
      }
    })
    
    document.addEventListener('keyup', (e) => {
      this.keys[e.code] = false
    })
    
    document.addEventListener('mousemove', (e) => {
      if (!this.isLocked) return
      
      this.euler.setFromQuaternion(this.camera.quaternion)
      this.euler.y -= e.movementX * this.mouseSensitivity
      this.euler.x -= e.movementY * this.mouseSensitivity
      this.euler.x = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, this.euler.x))
      this.camera.quaternion.setFromEuler(this.euler)
    })
    
    document.addEventListener('click', () => {
      if (!this.isLocked) {
        canvas.value.requestPointerLock()
      }
    })
    
    document.addEventListener('pointerlockchange', () => {
      this.isLocked = document.pointerLockElement === canvas.value
    })
  }

  update(deltaTime) {
    if (!this.isLocked) return

    this.direction.set(0, 0, 0)
    
    if (this.keys['KeyW']) this.direction.z -= 1
    if (this.keys['KeyS']) this.direction.z += 1
    if (this.keys['KeyA']) this.direction.x -= 1
    if (this.keys['KeyD']) this.direction.x += 1
    
    this.direction.normalize()
    
    this.checkGrounded()
    
    if (this.isInVehicle && this.vehicleTransform) {
      this.updateInVehicle(deltaTime)
    } else {
      this.updateInWorld(deltaTime)
    }
    
    this.updateCamera()
    this.updateDebugInfo()
  }

  checkGrounded() {
    const bodyPos = this.body.translation()
    const bodyVel = this.body.linvel()
    
    this.groundRayOrigin.set(bodyPos.x, bodyPos.y - 0.75, bodyPos.z)
    
    const ray = new RAPIER.Ray(this.groundRayOrigin, this.groundRayDir)
    const hit = world.castRay(ray, 0.5, true, RAPIER.QueryFilterFlags.EXCLUDE_SENSORS)
    
    this.isGrounded = hit !== null && (bodyVel.y <= 0.2 || hit.toi < 0.2)
    
    if (debugRayHelper) {
      const endPoint = this.groundRayOrigin.clone().add(this.groundRayDir.clone().multiplyScalar(0.5))
      debugRayHelper.geometry.setFromPoints([this.groundRayOrigin, endPoint])
      debugRayHelper.material.color.setHex(this.isGrounded ? 0x00ff00 : 0xff0000)
    }
  }

  updateCamera() {
    const bodyPos = this.body.translation()
    
    if (isThirdPerson) {
      const offset = new THREE.Vector3(0, 3, 5)
      offset.applyQuaternion(this.camera.quaternion)
      this.camera.position.set(bodyPos.x + offset.x, bodyPos.y + offset.y, bodyPos.z + offset.z)
    } else {
      this.camera.position.set(bodyPos.x, bodyPos.y + 0.8, bodyPos.z)
    }
  }

  updateDebugInfo() {
    const bodyPos = this.body.translation()
    const bodyVel = this.body.linvel()
    
    document.getElementById('position').textContent = `${bodyPos.x.toFixed(1)}, ${bodyPos.y.toFixed(1)}, ${bodyPos.z.toFixed(1)}`
    document.getElementById('velocity').textContent = `${bodyVel.x.toFixed(1)}, ${bodyVel.y.toFixed(1)}, ${bodyVel.z.toFixed(1)}`
    document.getElementById('grounded').textContent = this.isGrounded
    document.getElementById('in-vehicle').textContent = this.isInVehicle
    
    if (vehicleBody) {
      const vehicleRot = vehicleBody.rotation()
      const vehicleQuat = new THREE.Quaternion(vehicleRot.x, vehicleRot.y, vehicleRot.z, vehicleRot.w)
      
      const vehicleUp = new THREE.Vector3(0, 1, 0).applyQuaternion(vehicleQuat)
      const vehicleForward = new THREE.Vector3(0, 0, -1).applyQuaternion(vehicleQuat)
      
      document.getElementById('vehicle-up').textContent = `${vehicleUp.x.toFixed(2)}, ${vehicleUp.y.toFixed(2)}, ${vehicleUp.z.toFixed(2)}`
      document.getElementById('vehicle-forward').textContent = `${vehicleForward.x.toFixed(2)}, ${vehicleForward.y.toFixed(2)}, ${vehicleForward.z.toFixed(2)}`
    }
  }

  updateInWorld(deltaTime) {
    const cameraDirection = new THREE.Vector3()
    this.camera.getWorldDirection(cameraDirection)
    cameraDirection.y = 0
    cameraDirection.normalize()
    
    const cameraRight = new THREE.Vector3()
    cameraRight.crossVectors(cameraDirection, new THREE.Vector3(0, 1, 0))
    
    const moveDirection = new THREE.Vector3()
    moveDirection.addScaledVector(cameraDirection, -this.direction.z)
    moveDirection.addScaledVector(cameraRight, this.direction.x)
    
    const currentVel = this.body.linvel()
    const newVel = {
      x: moveDirection.x * this.moveSpeed,
      y: currentVel.y,
      z: moveDirection.z * this.moveSpeed
    }
    
    if (this.keys['Space'] && this.isGrounded) {
      newVel.y = this.jumpForce
    }
    
    this.body.setLinvel(newVel, true)
  }

  updateInVehicle(deltaTime) {
    if (!this.vehicleTransform) return
    
    const cameraDirection = new THREE.Vector3()
    this.camera.getWorldDirection(cameraDirection)
    cameraDirection.y = 0
    cameraDirection.normalize()
    
    const cameraRight = new THREE.Vector3()
    cameraRight.crossVectors(cameraDirection, new THREE.Vector3(0, 1, 0))
    
    const moveDirection = new THREE.Vector3()
    moveDirection.addScaledVector(cameraDirection, -this.direction.z)
    moveDirection.addScaledVector(cameraRight, this.direction.x)
    
    const currentVel = this.body.linvel()
    const vehicleVel = vehicleBody.linvel()
    
    const worldVel = {
      x: moveDirection.x * this.moveSpeed + vehicleVel.x,
      y: currentVel.y,
      z: moveDirection.z * this.moveSpeed + vehicleVel.z
    }
    
    if (this.keys['Space'] && this.isGrounded) {
      worldVel.y = this.jumpForce
    }
    
    this.body.setLinvel(worldVel, true)
  }

  enterVehicle(vehicleTransform) {
    this.isInVehicle = true
    this.vehicleTransform = vehicleTransform
  }

  exitVehicle() {
    this.isInVehicle = false
    this.vehicleTransform = null
  }
}

onMounted(async () => {
  await RAPIER.init()
  
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x87CEEB)
  
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 5, 10)
  
  renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  
  world = new RAPIER.World({ x: 0, y: -12, z: 0 })
  
  const ambientLight = new THREE.AmbientLight(0x404040, 0.6)
  scene.add(ambientLight)
  
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(50, 50, 50)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  directionalLight.shadow.camera.left = -100
  directionalLight.shadow.camera.right = 100
  directionalLight.shadow.camera.top = 100
  directionalLight.shadow.camera.bottom = -100
  directionalLight.shadow.camera.near = 0.1
  directionalLight.shadow.camera.far = 200
  scene.add(directionalLight)
  
  const groundGeometry = new THREE.PlaneGeometry(500, 500)
  const groundMaterial = new THREE.MeshLambertMaterial({ color: 0x90EE90 })
  const ground = new THREE.Mesh(groundGeometry, groundMaterial)
  ground.rotation.x = -Math.PI / 2
  ground.receiveShadow = true
  scene.add(ground)
  
  const groundColliderDesc = RAPIER.ColliderDesc.cuboid(250, 0.1, 250)
  world.createCollider(groundColliderDesc)
  
  const playerGeometry = new THREE.CapsuleGeometry(0.5, 1.5, 4, 8)
  const playerMaterial = new THREE.MeshLambertMaterial({ color: 0xff0000 })
  player = new THREE.Mesh(playerGeometry, playerMaterial)
  player.position.set(-20, 2, 0)
  player.castShadow = true
  scene.add(player)
  
  const playerRigidBodyDesc = RAPIER.RigidBodyDesc.dynamic()
  playerRigidBodyDesc.setTranslation(-20, 2, 0)
  playerRigidBodyDesc.lockRotations()
  playerBody = world.createRigidBody(playerRigidBodyDesc)
  
  const playerColliderDesc = RAPIER.ColliderDesc.capsule(0.75, 0.5)
  playerCollider = world.createCollider(playerColliderDesc, playerBody)
  
  playerController = new FPSController(camera, playerBody, world)
  
  const rayGeometry = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(), new THREE.Vector3()])
  const rayMaterial = new THREE.LineBasicMaterial({ color: 0xff0000 })
  debugRayHelper = new THREE.Line(rayGeometry, rayMaterial)
  scene.add(debugRayHelper)
  
  const loader = new GLTFLoader()
  loader.load('/Plane.glb', (gltf) => {
    vehicle = gltf.scene
    vehicle.position.set(10, 1, 0)
    scene.add(vehicle)
    
    const vehicleRigidBodyDesc = RAPIER.RigidBodyDesc.dynamic()
    vehicleRigidBodyDesc.setTranslation(10, 1, 0)
    vehicleBody = world.createRigidBody(vehicleRigidBodyDesc)
    
    let bodyMesh = null
    let floorMesh = null
    let bayMesh = null
    
    vehicle.traverse((child) => {
      if (child.isMesh) {
        switch (child.name) {
          case 'Body':
            bodyMesh = child
            break
          case 'Floor':
            floorMesh = child
            break
          case 'Bay':
            bayMesh = child
            break
        }
      }
    })
    
    if (bodyMesh) {
      const bodyGeometry = bodyMesh.geometry.clone()
      bodyGeometry.applyMatrix4(bodyMesh.matrixWorld)
      
      const vertices = bodyGeometry.attributes.position.array
      const indices = bodyGeometry.index ? bodyGeometry.index.array : null
      
      const bodyColliderDesc = RAPIER.ColliderDesc.trimesh(vertices, indices)
      world.createCollider(bodyColliderDesc, vehicleBody)
    }
    
    if (floorMesh) {
      const floorGeometry = floorMesh.geometry.clone()
      floorGeometry.applyMatrix4(floorMesh.matrixWorld)
      
      const vertices = floorGeometry.attributes.position.array
      const indices = floorGeometry.index ? floorGeometry.index.array : null
      
      const floorColliderDesc = RAPIER.ColliderDesc.trimesh(vertices, indices)
      world.createCollider(floorColliderDesc, vehicleBody)
      
      const debugFloorGeometry = new THREE.BufferGeometry()
      debugFloorGeometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3))
      if (floorGeometry.index) {
        debugFloorGeometry.setIndex(floorGeometry.index)
      }
      
      const debugFloorMaterial = new THREE.MeshBasicMaterial({ 
        color: 0x0000ff, 
        wireframe: true, 
        transparent: true, 
        opacity: 0.5,
        side: THREE.DoubleSide
      })
      floorDebugMesh = new THREE.Mesh(debugFloorGeometry, debugFloorMaterial)
      floorDebugMesh.position.copy(vehicle.position)
      floorDebugMesh.quaternion.copy(vehicle.quaternion)
      floorDebugMesh.scale.copy(vehicle.scale)
      scene.add(floorDebugMesh)
    }
    
    if (bayMesh) {
      console.log('Bay mesh found:', bayMesh.name)
      const bayGeometry = bayMesh.geometry.clone()
      
      const tempMatrix = new THREE.Matrix4()
      tempMatrix.copy(bayMesh.matrixWorld)
      tempMatrix.multiply(vehicle.matrix)
      bayGeometry.applyMatrix4(tempMatrix)
      
      const vertices = bayGeometry.attributes.position.array
      console.log('Bay vertices count:', vertices.length / 3)
      
      const bayColliderDesc = RAPIER.ColliderDesc.convexHull(vertices)
      if (bayColliderDesc) {
        console.log('Bay collider created successfully')
        bayColliderDesc.setSensor(true)
        const bayCollider = world.createCollider(bayColliderDesc, vehicleBody)
        interiorSensors.push(bayCollider)
        console.log('Bay sensor added to list, total sensors:', interiorSensors.length)
      } else {
        console.error('Failed to create bay convex hull collider')
      }
      
      const debugBayGeometry = new THREE.BufferGeometry()
      debugBayGeometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3))
      if (bayGeometry.index) {
        debugBayGeometry.setIndex(bayGeometry.index)
      }
      
      const debugBayMaterial = new THREE.MeshBasicMaterial({ 
        color: 0x00ff00, 
        wireframe: true, 
        transparent: true, 
        opacity: 0.3 
      })
      bayDebugMesh = new THREE.Mesh(debugBayGeometry, debugBayMaterial)
      bayDebugMesh.position.copy(vehicle.position)
      bayDebugMesh.quaternion.copy(vehicle.quaternion)
      bayDebugMesh.scale.copy(vehicle.scale)
      scene.add(bayDebugMesh)
      
      bayMesh.visible = false
    }
    
  }, undefined, (error) => {
    console.warn('GLTF loading failed, creating placeholder vehicle')
    
    const vehicleGeometry = new THREE.BoxGeometry(4, 2, 8)
    const vehicleMaterial = new THREE.MeshLambertMaterial({ color: 0x0000ff })
    vehicle = new THREE.Mesh(vehicleGeometry, vehicleMaterial)
    vehicle.position.set(10, 1, 0)
    vehicle.castShadow = true
    scene.add(vehicle)
    
    const vehicleRigidBodyDesc = RAPIER.RigidBodyDesc.dynamic()
    vehicleRigidBodyDesc.setTranslation(10, 1, 0)
    vehicleBody = world.createRigidBody(vehicleRigidBodyDesc)
    
    const vehicleColliderDesc = RAPIER.ColliderDesc.cuboid(2, 1, 4)
    world.createCollider(vehicleColliderDesc, vehicleBody)
    
    const interiorFloorDesc = RAPIER.ColliderDesc.cuboid(1.8, 0.1, 3.8)
    interiorFloorDesc.setTranslation(0, 0.5, 0)
    world.createCollider(interiorFloorDesc, vehicleBody)
    
    const interiorSensorDesc = RAPIER.ColliderDesc.cuboid(1.8, 1.5, 3.8)
    interiorSensorDesc.setTranslation(0, 1.5, 0)
    interiorSensorDesc.setSensor(true)
    const interiorSensor = world.createCollider(interiorSensorDesc, vehicleBody)
    interiorSensors.push(interiorSensor)
  })
  
  const clock = new THREE.Clock()
  
  function animate() {
    requestAnimationFrame(animate)
    
    const deltaTime = clock.getDelta()
    
    world.step()
    
    playerController.update(deltaTime)
    
    let playerInAnySensor = false
    let intersectionCount = 0
    
    world.intersectionsWith(playerCollider, (collider) => {
      intersectionCount++
      if (interiorSensors.includes(collider)) {
        playerInAnySensor = true
        if (!isPlayerInVehicle) {
          console.log('Player entering vehicle sensor')
          isPlayerInVehicle = true
          const vehiclePos = vehicleBody.translation()
          const vehicleRot = vehicleBody.rotation()
          const vehicleScale = new THREE.Vector3(1, 1, 1)
          
          playerController.enterVehicle({
            position: new THREE.Vector3(vehiclePos.x, vehiclePos.y, vehiclePos.z),
            quaternion: new THREE.Quaternion(vehicleRot.x, vehicleRot.y, vehicleRot.z, vehicleRot.w),
            scale: vehicleScale
          })
        }
      }
    })
    
    if (isPlayerInVehicle && !playerInAnySensor) {
      console.log('Player exiting vehicle sensor')
      isPlayerInVehicle = false
      playerController.exitVehicle()
    }
    
    document.getElementById('intersections').textContent = intersectionCount
    document.getElementById('sensors').textContent = interiorSensors.length
    
    if (bayDebugMesh) {
      bayDebugMesh.material.color.setHex(playerInAnySensor ? 0xff0000 : 0x00ff00)
      bayDebugMesh.material.opacity = playerInAnySensor ? 0.6 : 0.3
    }
    
    if (vehicle) {
      const vehiclePos = vehicleBody.translation()
      const vehicleRot = vehicleBody.rotation()
      vehicle.position.set(vehiclePos.x, vehiclePos.y, vehiclePos.z)
      vehicle.quaternion.set(vehicleRot.x, vehicleRot.y, vehicleRot.z, vehicleRot.w)
      
      if (bayDebugMesh) {
        bayDebugMesh.position.set(vehiclePos.x, vehiclePos.y, vehiclePos.z)
        bayDebugMesh.quaternion.set(vehicleRot.x, vehicleRot.y, vehicleRot.z, vehicleRot.w)
      }
      
      if (floorDebugMesh) {
        floorDebugMesh.position.set(vehiclePos.x, vehiclePos.y, vehiclePos.z)
        floorDebugMesh.quaternion.set(vehicleRot.x, vehicleRot.y, vehicleRot.z, vehicleRot.w)
      }
    }
    
    const playerPos = playerBody.translation()
    player.position.set(playerPos.x, playerPos.y, playerPos.z)
    
    renderer.render(scene, camera)
  }
  
  animate()
  
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  })
})
</script>

<style>
:global(*) {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

:global(body) {
  overflow: hidden;
  background: #000;
  font-family: Arial, sans-serif;
}

#game-container {
  position: relative;
  width: 100vw;
  height: 100vh;
}

#crosshair {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 4px;
  height: 4px;
  background: white;
  border-radius: 50%;
  pointer-events: none;
  z-index: 1000;
}

#ui {
  position: absolute;
  top: 20px;
  left: 20px;
  color: white;
  font-size: 14px;
  z-index: 1000;
  background: rgba(0, 0, 0, 0.7);
  padding: 15px;
  border-radius: 5px;
  font-family: monospace;
}

#debug-info {
  margin-top: 10px;
  font-size: 12px;
  line-height: 1.4;
}

:global(canvas) {
  display: block;
}
</style>