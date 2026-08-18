<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'

const canvas = ref(null)
let ctx = null
let resizeTimer = null

/* ---------- deterministic randomness ---------- */
function mulberry32 (seed) {
  return function () {
    seed |= 0; seed = (seed + 0x6D2B79F5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}

const hillBack = [[0, 0.65], [0.2, 0.62], [0.5, 0.66], [0.8, 0.63], [1, 0.65]]
const hillMid = [[0, 0.75], [0.3, 0.77], [0.6, 0.74], [0.85, 0.76], [1, 0.73]]
const hillFront = [[0, 0.88], [0.25, 0.85], [0.5, 0.89], [0.75, 0.86], [1, 0.88]]

function ridgeYAt (pts, t) {
  for (let i = 1; i < pts.length; i++) {
    if (t <= pts[i][0]) {
      const [x0, y0] = pts[i - 1]
      const [x1, y1] = pts[i]
      const localT = (t - x0) / (x1 - x0 || 1)
      return y0 + (y1 - y0) * localT
    }
  }
  return pts[pts.length - 1][1]
}

/* ---------- Fractal Mountain Generator ---------- */
function midpointDisplacement(pts, roughness, rand) {
  let newPts = []
  for(let i=0; i < pts.length - 1; i++) {
    newPts.push(pts[i])
    let midX = (pts[i][0] + pts[i+1][0]) / 2
    let midY = (pts[i][1] + pts[i+1][1]) / 2
    midY += (rand() - 0.5) * roughness 
    newPts.push([midX, midY])
  }
  newPts.push(pts[pts.length - 1])
  return newPts
}

function makeFramingMountain(seed, yEdge, yMid, peakAmp) {
  const rand = mulberry32(seed)
  let pts = [
    [0, yEdge + rand() * 0.1],
    [0.15, yEdge + (yMid - yEdge) * 0.2 - rand() * peakAmp],
    [0.5, yMid],
    [0.85, yEdge + (yMid - yEdge) * 0.2 - rand() * peakAmp],
    [1, yEdge + rand() * 0.1]
  ]
  
  pts = midpointDisplacement(pts, 0.08, rand)
  pts = midpointDisplacement(pts, 0.05, rand)
  pts = midpointDisplacement(pts, 0.03, rand)
  pts = midpointDisplacement(pts, 0.015, rand)
  pts = midpointDisplacement(pts, 0.005, rand)
  
  return pts
}

function makeForest (seed, count, sizeRange) {
  const rand = mulberry32(seed)
  const trees = []
  for (let i = 0; i < count; i++) {
    trees.push({
      t: rand(),
      size: sizeRange[0] + rand() * (sizeRange[1] - sizeRange[0]),
      jitter: (rand() - 0.5) * 0.02
    })
  }
  return trees.sort((a, b) => a.t - b.t)
}

function makeBlooms(seed, clusterCount, flowersPerCluster, surfaceRidge, treesToAvoid) {
  const rand = mulberry32(seed)
  const colors = ['#d13b70', '#e6739f', '#8a4988', '#f2c14e', '#bd3c53', '#fdfdfd']
  const generated = []
  
  for(let c=0; c < clusterCount; c++) {
    let cx = rand() 
    let cMinY = ridgeYAt(surfaceRidge, cx) + 0.05
    if (cMinY > 0.94) cMinY = 0.94
    let cy = cMinY + rand() * (1.0 - cMinY)

    for(let i=0; i < flowersPerCluster; i++) {
      let ux = rand() + rand() + rand() - 1.5
      let uy = rand() + rand() + rand() - 1.5
      
      // Wider horizontal spread for more natural meadow bands
      let x = cx + ux * 0.12
      if (x < 0 || x > 1) continue
      
      // Only avoid the very center trunk of the trees so flowers can sit under the canopy
      let isOverlappingTree = treesToAvoid.some(tree => {
        let trunkWidth = 0.025 
        return Math.abs(x - tree.t) < trunkWidth
      })
      
      if (isOverlappingTree) continue

      let minY = ridgeYAt(surfaceRidge, x) + 0.01
      // Tighter vertical spread to keep them grouped nicely on the hills
      let y = cy + uy * 0.05
      if (y < minY) y = minY + rand() * 0.03
      if (y > 0.98) y = 0.98

      let size = 1.0 + rand() * 2.2
      let color = colors[Math.floor(rand() * colors.length)]
      let rot = rand() * Math.PI * 2
      generated.push([x, y, color, size, rot])
    }
  }
  return generated
}

const mountainBack = makeFramingMountain(7, 0.2, 0.7, 0.35)
const mountainFront = makeFramingMountain(13, 0.3, 0.85, 0.45)

const forestBack = makeForest(21, 35, [0.06, 0.09])
const forestMid = makeForest(34, 45, [0.08, 0.13])
const forestFront = makeForest(55, 55, [0.12, 0.18])

// 45 clusters of 20 flowers to create dense, wide patches across the bottom
const blooms = makeBlooms(42, 45, 20, hillMid, forestMid)

function drawTexturedMountain (w, h, pts, lightColor, shadowColor) {
  ctx.beginPath()
  ctx.moveTo(pts[0][0] * w, h)
  pts.forEach(([x, y]) => ctx.lineTo(x * w, y * h))
  ctx.lineTo(pts[pts.length - 1][0] * w, h)
  ctx.closePath()
  ctx.fillStyle = lightColor
  ctx.fill()

  ctx.fillStyle = shadowColor
  ctx.beginPath()
  for (let i = 0; i < pts.length - 1; i++) {
    let p1 = pts[i]
    let p2 = pts[i+1]
    if (p2[1] > p1[1]) { 
      ctx.moveTo(p1[0] * w, p1[1] * h)
      ctx.lineTo(p2[0] * w, p2[1] * h)
      ctx.lineTo(p2[0] * w - w * 0.005, p2[1] * h + h * 0.06)
      ctx.lineTo(p1[0] * w - w * 0.005, p1[1] * h + h * 0.06)
    }
  }
  ctx.fill()
}

function drawSnowCaps (w, h, pts, threshold, seed) {
  const rand = mulberry32(seed)
  ctx.fillStyle = 'rgba(253, 245, 240, 0.95)'
  
  ctx.beginPath()
  ctx.moveTo(pts[0][0] * w, pts[0][1] * h)
  
  for (let i = 1; i < pts.length; i++) {
    ctx.lineTo(pts[i][0] * w, pts[i][1] * h)
  }

  for (let i = pts.length - 1; i >= 0; i--) {
    let [x, y] = pts[i]
    if (y < threshold) {
      let depth = threshold - y
      let drop = depth * (0.4 + rand() * 0.6)
      ctx.lineTo(x * w, (y + drop) * h)
    } else {
      ctx.lineTo(x * w, y * h)
    }
  }
  ctx.closePath()
  ctx.fill()
}

function drawHillFill (w, h, pts, color) {
  ctx.beginPath()
  ctx.moveTo(pts[0][0] * w, h)
  ctx.lineTo(pts[0][0] * w, pts[0][1] * h)
  for (let i = 1; i < pts.length; i++) {
    const x = pts[i][0] * w, y = pts[i][1] * h
    const px = pts[i - 1][0] * w, py = pts[i - 1][1] * h
    ctx.quadraticCurveTo((px + x) / 2, Math.min(py, y), x, y)
  }
  ctx.lineTo(pts[pts.length - 1][0] * w, h)
  ctx.closePath()
  ctx.fillStyle = color
  ctx.fill()
}

function drawPine (x, yBase, hgt, color) {
  const w = hgt * 0.55
  ctx.fillStyle = color
  ctx.fillRect(x - hgt * 0.03, yBase - hgt * 0.1, hgt * 0.06, hgt * 0.1)
  ctx.beginPath()
  ctx.moveTo(x, yBase - hgt * 0.72)
  ctx.lineTo(x - w / 2, yBase)
  ctx.lineTo(x + w / 2, yBase)
  ctx.closePath(); ctx.fill()
  ctx.beginPath()
  ctx.moveTo(x, yBase - hgt)
  ctx.lineTo(x - w * 0.36, yBase - hgt * 0.38)
  ctx.lineTo(x + w * 0.36, yBase - hgt * 0.38)
  ctx.closePath(); ctx.fill()
}

function drawForest (w, h, ridgePts, trees, colors) {
  trees.forEach((tree, i) => {
    const x = tree.t * w
    const yBase = (ridgeYAt(ridgePts, tree.t) + tree.jitter) * h
    const hgt = tree.size * h
    const color = colors[i % colors.length]
    drawPine(x, yBase, hgt, color)
  })
}

function draw () {
  if (!ctx) return
  const dpr = Math.min(window.devicePixelRatio || 1, 2)
  const w = window.innerWidth, h = window.innerHeight
  canvas.value.width = w * dpr
  canvas.value.height = h * dpr
  canvas.value.style.width = w + 'px'
  canvas.value.style.height = h + 'px'
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)

  const sky = ctx.createLinearGradient(0, 0, 0, h)
  sky.addColorStop(0, '#bea9de')
  sky.addColorStop(0.3, '#f1a8b3')
  sky.addColorStop(0.6, '#fce4b4')
  sky.addColorStop(1, '#d8e2c3')
  ctx.fillStyle = sky
  ctx.fillRect(0, 0, w, h)

  const gx = w * 0.15, gy = h * 0.25, gr = Math.max(w, h) * 0.4
  const glow = ctx.createRadialGradient(gx, gy, 0, gx, gy, gr)
  glow.addColorStop(0, 'rgba(255, 230, 180, 0.7)')
  glow.addColorStop(1, 'rgba(255, 230, 180, 0)')
  ctx.fillStyle = glow
  ctx.beginPath(); ctx.arc(gx, gy, gr, 0, Math.PI * 2); ctx.fill()

  drawTexturedMountain(w, h, mountainBack, '#6e6773', '#5a545e')
  drawSnowCaps(w, h, mountainBack, 0.52, 42) // Reduced from 0.65 to 0.52

  drawTexturedMountain(w, h, mountainFront, '#4a474f', '#36343a')
  drawSnowCaps(w, h, mountainFront, 0.62, 84) // Reduced from 0.75 to 0.62

  drawHillFill(w, h, hillBack, '#5c6b45')
  drawForest(w, h, hillBack, forestBack, ['#2a4225', '#1e331a'])

  drawHillFill(w, h, hillMid, '#44542d')
  drawForest(w, h, hillMid, forestMid, ['#182e16', '#253d21', '#3b4d24'])

  drawHillFill(w, h, hillFront, '#31401f')
  drawForest(w, h, hillFront, forestFront, ['#11210f', '#0c170a', '#182915'])

  blooms.forEach((b) => {
    ctx.save()
    ctx.translate(b[0] * w, b[1] * h)
    ctx.rotate(b[4])
    ctx.globalAlpha = 0.95
    
    const size = b[3]
    
    ctx.fillStyle = b[2]
    for(let i=0; i<5; i++) {
      ctx.beginPath()
      ctx.ellipse(0, size * 0.7, size * 0.45, size * 0.9, 0, 0, Math.PI * 2)
      ctx.fill()
      ctx.rotate((Math.PI * 2) / 5)
    }
    
    ctx.fillStyle = '#fce4b4'
    ctx.beginPath()
    ctx.arc(0, 0, size * 0.4, 0, Math.PI * 2)
    ctx.fill()
    
    ctx.restore()
  })

  const vignette = ctx.createRadialGradient(w / 2, h / 2, Math.min(w, h) * 0.35, w / 2, h / 2, Math.max(w, h) * 0.8)
  vignette.addColorStop(0, 'rgba(0,0,0,0)')
  vignette.addColorStop(1, 'rgba(15,10,20,.25)')
  ctx.fillStyle = vignette
  ctx.fillRect(0, 0, w, h)
}

function onResize () {
  clearTimeout(resizeTimer)
  resizeTimer = setTimeout(draw, 120)
}

onMounted(() => {
  ctx = canvas.value.getContext('2d')
  draw()
  window.addEventListener('resize', onResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize)
  clearTimeout(resizeTimer)
})
</script>

<template>
  <canvas ref="canvas" class="garden" aria-hidden="true"></canvas>
</template>

<style scoped>
.garden{
  position:fixed; inset:0; width:100vw; height:100vh;
  z-index:0; pointer-events:none;
}
</style>