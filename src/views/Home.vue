<template>
  <div class="container" id="container" ref="container" @click="onMouseClick">
    <div v-if="mode == 'design'" class="compose-button">
      <el-button @click.stop.prevent="composeMesh">绑定</el-button>
    </div>
    <div v-if="mode == 'design'" class="discompose-button">
      <el-button @click.stop.prevent="discomposeGroup">解绑</el-button>
    </div>
    <div class="route-button">
      <el-button @click.stop.prevent="routes">route</el-button>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
// import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js'
// import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js'
// import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass.js'
// import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass.js'
// import { FXAAShader } from 'three/examples/jsm/shaders/FXAAShader.js'
import Stats from 'three/examples/jsm/libs/stats.module.js'
import { v4 as uuidv4 } from 'uuid'

export default {
  name: 'Home',
  components: {},
  data() {
    return {
      group: null,
      objects: [],
      effectFXAA: null,
      // outlinePass: null,
      pixelRatio: null,
      // fxaaPass: null,
      mouse: new THREE.Vector2(),
      dbmouse: new THREE.Vector2(),
      raycaster: new THREE.Raycaster(),
      selectedObjects: [],
      // composer: null,
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
    this.init()
    if (this.renderer != null && this.$refs.container != null) {
      this.animate()
    }
    window.addEventListener('resize', this.onWindowResize, false)
  },
  beforeDestroy() {
    this.resetParams()
    window.removeEventListener('resize', this.onWindowResize, false) //这里是resize整个窗口的监听
    console.log(this.renderer)
    console.log(this.scene)
  },
  destroyed() {
    console.log(this.atomicArr)
  },
  watch: {},
  methods: {
    init() {
      this.initScene()
      this.initCamera()
      this.initRenderer()
      // this.initGeometry()
      this.initLoader()
      this.initLight()
      this.initControls()
      // this.initComposer()
      this.initStats()
    },

    resetParams() {
      this.raycaster = null
      this.camera = null
      this.scene.clear()
      window.cancelAnimationFrame(this.animationFrame)
      console.log(this.scene)
      this.renderer.dispose()
      this.renderer.forceContextLoss()
      this.renderer.content = null
      this.renderer = null
      for (let i = 0; i < this.atomicArr.length; i++) {
        this.atomicArr[i].geometry.dispose()
        this.atomicArr[i].material.dispose()
        // this.atomicArr[i].geometry = null
        // this.atomicArr[i].material = null
        console.log(this.atomicArr[i])
      }
      this.controls.dispose()
      this.controls = null
      this.atomicArr = null
      this.groupMap = null
      // console.log(this.groupMap)
      this.light = null
      this.loader = null
      // console.log('all stuffs reset')
    },

    //初始化透视摄像机，此外还有正交摄像机
    //那是不是甚至可以模拟摄像机，比如说在环境中加入n个摄像机，每个摄像机有其vector3坐标，然后lookat某个点，然后双击后切换到该摄像机看到的事件？
    initCamera() {
      this.camera = new THREE.PerspectiveCamera(65, this.width / this.height, 0.1, 10000)
      this.camera.position.set(5, 5, 30)
    },

    //初始化scene
    initScene() {
      this.scene = new THREE.Scene()
    },
    //初始化光照
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
      let geometry = new THREE.BoxGeometry(5, 5, 5)
      let material = new THREE.MeshNormalMaterial()
      let mesh = new THREE.Mesh(geometry, material)
      this.atomicArr.push(mesh)
      this.scene.add(mesh)
      window.cancelAnimationFrame(this.animationFrame)
      this.animationFrame = window.requestAnimationFrame(this.animate)
    },
    //加载helmet
    initLoader() {
      let loader = new GLTFLoader()
      loader.load(
        '/flight_helmet/FlightHelmet.gltf',
        object => {
          console.log(object)
          console.log(object.scene)
          console.log(object.scene.children[0].children.length)
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
          console.log(object.parser)
          object.parser.cache.removeAll()
          object.parser = null
        },
        onprogress,
        function (err) {
          console.log(err)
        }
      )
    },
    // initComposer() {
    //   // if (this.renderer != null) {
    //   this.composer = new EffectComposer(this.renderer)
    //   let renderPass = new RenderPass(this.scene, this.camera)
    //   this.fxaaPass = new ShaderPass(FXAAShader)
    //   const pixelRatio = this.renderer.getPixelRatio()
    //   this.fxaaPass.material.uniforms['resolution'].value.x = 1 / (this.width * pixelRatio)
    //   this.fxaaPass.material.uniforms['resolution'].value.y = 1 / (this.height * pixelRatio)
    //   this.composer.addPass(renderPass)
    //   this.composer.addPass(this.fxaaPass)
    //   this.outlinePass = new OutlinePass(new THREE.Vector2(this.width, this.height), this.scene, this.camera)
    //   this.outlinePass.edgeStrength = 3 //包围线浓度
    //   this.outlinePass.edgeGlow = 1 //边缘线范围
    //   this.outlinePass.edgeThickness = 1 //边缘线浓度
    //   this.outlinePass.pulsePeriod = 2 //包围线闪烁频率
    //   this.outlinePass.visibleEdgeColor.set('#00ffff') //包围线颜色
    //   this.outlinePass.hiddenEdgeColor.set('#190a05') //被遮挡的边界线颜色
    //   this.outlinePass.renderToScreen = true
    //   this.composer.addPass(this.outlinePass)
    //   // }
    // },

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

    onMouseClick(event) {
      console.log(this.scene)
      console.log(event.offsetX)
      console.log(this.$refs.container.getBoundingClientRect())
      this.mouse.x = (event.offsetX / (this.width * (1 / 1))) * 2 - 1
      this.mouse.y = -(event.offsetY / (this.height * (1 / 1))) * 2 + 1

      this.raycaster.setFromCamera(this.mouse, this.camera)

      var intersects = this.raycaster.intersectObjects(this.scene.children, true)

      //当选中了确切的物体时
      if (intersects.length > 0) {
        console.log(intersects)
        let tempStore = intersects[0].object
        //推回至最上层的父结点，选中最上层的这个父结点
        // while (tempStore.parent.type != 'Scene') {
        //   tempStore = tempStore.parent
        // }
        let tempuuid = tempStore.userData.uuid
        //选中的物体uuid == -1时
        if (tempuuid == -1) {
          //如果当前未选中该物体
          if (this.selectedObjects.indexOf(tempStore) < 0) {
            // let tempGroupArr = this.groupMap.get(tempuuid)
            this.selectedObjects.push(tempStore)
          }
        }
        if (tempuuid != -1) {
          //如果当前未选中该物体
          if (this.selectedObjects.indexOf(tempStore) < 0) {
            let tempGroupArr = this.groupMap.get(tempuuid)
            for (let i = 0; i < tempGroupArr.length; i++) {
              this.selectedObjects.push(tempGroupArr[i])
            }
          }
          //如果当前已选中该物体
          else if (this.selectedObjects.indexOf(tempStore) >= 0) {
            let tempGroupArr = this.groupMap.get(tempuuid)
            this.selectedObjects.splice(this.selectedObjects.indexOf(tempStore), 1)
            for (let i = 0; i < tempGroupArr.length; i++) {
              this.selectedObjects.push(tempGroupArr[i])
            }
          }
        }
        console.log(this.selectedObjects)
        // this.outlinePass.selectedObjects = this.selectedObjects
      }

      if (intersects.length == 0) {
        console.log('nothing selected')
        this.selectedObjects = []
        // this.outlinePass.selectedObjects = []
      }
    },
    /**
     * 把需要组合的mesh的userData.uuid赋为同一值，通过该值将所有的mesh联系起来
     */
    composeMesh() {
      console.log(this.selectedObjects)
      //如果被选中物体有2个及以上
      if (this.selectedObjects.length > 1) {
        //新建一个group
        let tempGroup = new THREE.Group()
        //从场景中删除所有被选中的物体，将被删除的物体添加入组中
        //2021.4.6，不再删除场景中被选中的物体，转而将被选中的物体设置uuid，作为compose的结果
        let tempuuid = uuidv4()
        let tempGroupArr = []
        //浅拷贝一份当前选中的东西的引用,并且把uuid赋给每个selectedObject，解绑时把uuid赋为-1
        for (let i = 0; i < this.selectedObjects.length; i++) {
          tempGroupArr.push(this.selectedObjects[i])
          this.selectedObjects[i].userData.uuid = tempuuid
        }
        //在map中添加当前tempGroupArr，方便查找
        this.groupMap.set(tempuuid, tempGroupArr)
        this.selectedObjects = []
        // this.outlinePass.selectedObjects = []
      }
    },
    /**
     * mesh.userData.uuid 值赋为-1，解绑
     */
    discomposeGroup() {
      //当选中的mesh是>=2个时才有删除必要
      if (this.selectedObjects.length > 1) {
        let tempSet = new Set()
        for (let i = 0; i < this.selectedObjects.length; i++) {
          tempSet.add(this.selectedObjects[i].userData.uuid)
          this.selectedObjects[i].userData.uuid = -1
        }
        for (let arruuid of tempSet) {
          this.groupMap.delete(arruuid)
        }
      }
      this.selectedObjects = []
      // this.outlinePass.selectedObjects = []
    },
    //深度遍历树，把所有叶子结点添加入数组中
    DFS(node, nodeList) {
      // console.log(node)
      if (node) {
        if (node.children.length == 0) {
          nodeList.push(node)
          // this.geometry.push(node.geometry)
          // this.material.push(node.material)
          // node.geometry.dispose()
          // node.material.aoMap.dispose()
          // node.material.dispose()
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
      console.log(1)
      console.log(this.$route.path)
      if (this.$route.path == '/') {
        this.$router.push('/about')
        console.log(2)
      } else if (this.$route.path == '/about') {
        this.$router.push('/')
        console.log(3)
      }
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

.compose-button {
  z-index: 20000;
  position: absolute;
  left: 100px;
}

.discompose-button {
  z-index: 20000;
  position: absolute;
  left: 200px;
}

.route-button {
  z-index: 20000;
  position: absolute;
  left: 300px;
}
</style>
