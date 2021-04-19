<template>
  <div class="container" id="container" ref="container">
    <div class="route-button">
      <el-button @click.stop.prevent="routes">route</el-button>
    </div>
    <div class="dispose-button">
      <el-button @click.stop.prevent="disposeTest">dispose</el-button>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import Stats from 'three/examples/jsm/libs/stats.module.js'
import { v4 as uuidv4 } from 'uuid'

export default {
  name: 'About',
  components: {},
  data() {
    return {
      group: null,
      objects: [],
      pixelRatio: null,
      selectedObjects: [],
      stats: null,
      animationFrame: null,
      directionalLight: null,
      ambientLight: null,
      atomicArr: [],
      width: window.innerWidth,
      height: window.innerHeight,
      mode: 'design',

      material: [],
      geometry: [],
      groupMap: new Map()
    }
  },
  props: {},
  computed: {},

  created() {},
  mounted() {
    this.flag = true
    this.camera = null
    this.scene = null
    this.renderer = null
    this.light = []
    this.controls = null
    this.loader = new GLTFLoader()
    this.init()
    if (this.renderer != null && this.$refs.container != null) {
      this.animate()
    }
    window.addEventListener('resize', this.onWindowResize, false)
  },
  beforeDestroy() {
    this.resetParams()
    window.removeEventListener('resize', this.onWindowResize, false) //这里是resize整个窗口的监听
  },
  destroyed() {
    // console.log(this.atomicArr)
  },
  watch: {},
  methods: {
    init() {
      this.initScene()
      this.initCamera()
      this.initRenderer()
      // for (let i = 0; i < 500; i++) {
      //   // this.disposeTest()
      //   this.initGeometry()
      // }
      this.initLoader()
      this.initLight()
      this.initControls()
      this.initStats()
    },

    resetParams() {
      window.cancelAnimationFrame(this.animationFrame)
      this.disposeTest()
      this.raycaster = null
      this.camera = null
      this.renderer.dispose()
      this.renderer.forceContextLoss()
      this.renderer.content = null
      this.renderer = null
      this.controls.dispose()
      this.controls = null
      this.atomicArr = null
      this.groupMap = null
      this.light = null
      this.loader = null
    },

    initCamera() {
      this.camera = new THREE.PerspectiveCamera(65, this.width / this.height, 0.1, 10000)
      this.camera.position.set(-1, 1, 1)
    },
    initScene() {
      this.scene = new THREE.Scene()
    },
    initLight() {
      // let directionalLight = track(new THREE.DirectionalLight(0xffffff, 2)) //平行光源
      let directionalLight = new THREE.DirectionalLight(0xffffff, 2)
      directionalLight.color.setHSL(0.1, 1, 0.95)
      directionalLight.position.set(0, 200, 200).normalize()
      this.light.push(directionalLight)

      // let ambient = track(new THREE.AmbientLight(0xffffff, 1.5)) //环境光源，提供基础亮度
      let ambient = new THREE.AmbientLight(0xffffff, 1.5)
      ambient.position.set(100, 100, 100)
      this.light.push(ambient)
      this.addLight()
    },
    removeLight() {
      for (let i = 0; i < this.light.length; i++) {
        this.scene.remove(this.light[i])
      }
    },
    addLight() {
      for (let i = 0; i < this.light.length; i++) {
        this.scene.add(this.light[i])
      }
    },
    initRenderer() {
      this.renderer = new THREE.WebGLRenderer({
        antialias: true
      })
      this.renderer.shadowMap.enabled = true
      this.renderer.setSize(this.width, this.height)
      this.renderer.setClearColor(0xffaaaa, 1.0)
      this.$refs.container.appendChild(this.renderer.domElement)
      // }
    },
    initControls() {
      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true // an animation loop is required when either damping or auto-rotation are enabled
      this.controls.dampingFactor = 0.2 //惯性旋转，默认0.25
      this.controls.screenSpacePanning = false
      this.controls.minDistance = 1
      this.controls.maxDistance = 1500
      this.controls.maxPolarAngle = Math.PI / 2
      //控制垂直旋转的角度
    },
    initGeometry() {
      let geometry = new THREE.BoxGeometry(1, 1, 1)
      let material = new THREE.MeshNormalMaterial()
      let mesh = new THREE.Mesh(geometry, material)
      this.atomicArr.push(mesh)
      this.scene.add(mesh)
    },
    //加载helmet
    initLoader() {
      // let loader = new GLTFLoader()
      // this.disposeTest()
      this.loader.load(
        '/flight_helmet/FlightHelmet.gltf',
        object => {
          // object.scene.traverse(obj => {
          //   if (obj.type === 'Mesh') {
          //     this.atomicArr.push(obj)
          //     // this.scene.add(obj)
          //   }
          // })
          // for (let i = 0; i < this.atomicArr.length; i++) {
          //   this.scene.add(this.atomicArr[i])
          // }
          this.DFS(object.scene, this.atomicArr)
          for (let i = 0; i < this.atomicArr.length; i++) {
            this.scene.add(this.atomicArr[i])
          }
          object.parser.cache.removeAll()
          object.parser = null
          object = null
        },
        onprogress,
        function(err) {
          // console.log(err)
        }
      )
    },
    loadAnotherObject() {
      // let loader = new GLTFLoader()
      this.loader.load(
        '/lantern/Lantern.gltf',
        // '/flight_helmet/FlightHelmet.gltf',
        object => {
          for (let i = 0; i < this.atomicArr.length; i++) {
            this.atomicArr[i].geometry.dispose()
            this.atomicArr[i].material.dispose()
          }
          this.DFS(object.scene, this.atomicArr)
          for (let i = 0; i < this.atomicArr.length; i++) {
            // this.atomicArr[i].userData.workflowArr = this.workflowArr
            if (this.atomicArr[i].userData.uuid == undefined) {
              this.atomicArr[i].userData.uuid = -1
            }
            // this.atomicArr[i] = track(this.atomicArr[i])
            this.scene.add(this.atomicArr[i])
          }
          let tempuuidSet = new Set()
          //初始化groupmap
          //把所有uuid丢进set里
          for (let i = 0; i < this.atomicArr.length; i++) {
            tempuuidSet.add(this.atomicArr[i].userData.uuid)
          }
          //经过去重的uuid创建为map
          tempuuidSet.delete(-1)
          for (let i of tempuuidSet) {
            this.groupMap.set(i, [])
          }
          for (let i = 0; i < this.atomicArr.length; i++) {
            if (this.atomicArr[i].userData.uuid != -1) {
              this.groupMap.get(this.atomicArr[i].userData.uuid).push(this.atomicArr[i])
            }
          }
          // console.log(object.parser)
          object.parser.cache.removeAll()
          object.parser = null
        },
        onprogress,
        function(err) {
          // console.log(err)
        }
      )
    },
    initStats() {
      // if (this.$refs.container != undefined) {
      this.stats = new Stats()
      this.stats.domElement.style.position = 'absolute'
      this.stats.domElement.style.left = '0px'
      this.stats.domElement.style.top = '0px'
      // document.body.appendChild(this.stats.domElement)
      this.$refs.container.appendChild(this.stats.domElement)
      // }
    },
    update() {
      this.controls.update()
      this.stats.update()
    },

    animate() {
      this.render()
      this.animationFrame = window.requestAnimationFrame(this.animate)
      this.update()
      // this.composer.render()
    },

    render() {
      this.renderer.render(this.scene, this.camera)
    },

    onWindowResize() {
      this.height = window.innerHeight
      this.width = window.innerWidth
      this.camera.aspect = this.width / this.height
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(this.width, this.height)
      this.render()
    },

    //DFS tree, add all leaf node into atmoicArr
    DFS(node, nodeList) {
      // console.log(node)
      if (node) {
        if (node.children.length == 0) {
          nodeList.push(node)
        } else if (node.children.length > 0) {
          let children = node.children
          for (var i = 0; i < children.length; i++) {
            this.DFS(children[i], nodeList)
          }
        }
      }
      return nodeList
    },
    routes() {
      // console.log(1)
      // console.log(this.$route.path)
      if (this.$route.path == '/') {
        this.$router.push('/about')
        // console.log(2)
      } else if (this.$route.path == '/about') {
        this.$router.push('/')
        // console.log(3)
      }
    },
    disposeTest() {
      console.log(this.renderer.info)
      for (let i = 0; i < this.atomicArr.length; i++) {
        this.disposeMesh(this.scene, this.atomicArr[i])
      }
      //the most fucking critical code is to set this.atomicArr = null
      this.atomicArr = null
      this.atomicArr = []
      console.log(this.renderer.info)
    },
    disposeMesh(parent, child) {
      if (child.children.length) {
        const arr = child.children.filter(x => x)
        arr.forEach(a => {
          this.disposeMesh(child, a)
        })
      }
      if (child instanceof THREE.Mesh || child instanceof THREE.Line) {
        if (child.material.map) child.material.map.dispose()
        child.material.dispose()
        child.geometry.dispose()
      } else if (child.material) {
        child.material.dispose()
      }
      child.remove()
      parent.remove(child)
    }
  }
}
</script>

<style>
.container {
  width: 100%;
  height: 100%;
}

.back-button {
  z-index: 20000;
  position: absolute;
}

.dispose-button {
  z-index: 20000;
  position: absolute;
  left: 100px;
}

/* .discompose-button {
  z-index: 20000;
  position: absolute;
  left: 200px;
} */

.route-button {
  z-index: 20000;
  position: absolute;
  left: 300px;
}
</style>
