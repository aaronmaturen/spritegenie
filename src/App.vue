<script setup>
import { ref, onMounted, computed, onUnmounted } from 'vue'
import GIF from 'gif.js'
import { parseGIF, decompressFrames } from 'gifuct-js'

const canvasRef = ref(null)
const previewCanvasRef = ref(null)
const fileInputRef = ref(null)
const uploadedImage = ref(null)
const imagePreviewUrl = ref(null)

const frameWidth = ref(64)
const frameHeight = ref(64)
const frameCount = ref(12)
const framesPerRow = ref(6)
const columnGap = ref(0)
const rowGap = ref(0)
const frameDelay = ref(100)
const loopCount = ref(0)
const backgroundColor = ref('#FFFFFF')
const transparentBackground = ref(false)

const cropTop = ref(0)
const cropBottom = ref(0)
const cropLeft = ref(0)
const cropRight = ref(0)

const reduceJitter = ref(false)
const frameBounds = ref(new Map())

const currentFrame = ref(0)
const isPlaying = ref(false)
const animationInterval = ref(null)
const isGenerating = ref(false)
const generationProgress = ref(0)

const canvasWidth = computed(() => {
  if (!canvasRef.value) return 800
  return canvasRef.value.offsetWidth
})

const canvasHeight = computed(() => {
  if (!canvasRef.value) return 600
  return canvasRef.value.offsetHeight
})

const detectBorders = () => {
  if (!uploadedImage.value) return

  // Create a temporary canvas to analyze the entire image
  const tempCanvas = document.createElement('canvas')
  const ctx = tempCanvas.getContext('2d')
  tempCanvas.width = uploadedImage.value.width
  tempCanvas.height = uploadedImage.value.height
  ctx.drawImage(uploadedImage.value, 0, 0)

  const imageData = ctx.getImageData(0, 0, tempCanvas.width, tempCanvas.height)
  const data = imageData.data
  const width = imageData.width
  const height = imageData.height

  // Function to check if a pixel is transparent or nearly transparent
  const isTransparent = (idx) => data[idx + 3] < 10

  // Function to check if a row is empty (all transparent or uniform color)
  const isRowEmpty = (y) => {
    let transparentCount = 0
    let firstNonTransparent = null

    for (let x = 0; x < width; x++) {
      const idx = (y * width + x) * 4
      if (isTransparent(idx)) {
        transparentCount++
      } else if (firstNonTransparent === null) {
        firstNonTransparent = [data[idx], data[idx + 1], data[idx + 2]]
      } else {
        // Check if this pixel matches the first non-transparent pixel
        const colorDiff = Math.abs(data[idx] - firstNonTransparent[0]) +
                         Math.abs(data[idx + 1] - firstNonTransparent[1]) +
                         Math.abs(data[idx + 2] - firstNonTransparent[2])
        if (colorDiff > 30) return false // Not uniform
      }
    }

    return transparentCount > width * 0.95 || transparentCount === 0
  }

  // Function to check if a column is empty
  const isColumnEmpty = (x) => {
    let transparentCount = 0
    let firstNonTransparent = null

    for (let y = 0; y < height; y++) {
      const idx = (y * width + x) * 4
      if (isTransparent(idx)) {
        transparentCount++
      } else if (firstNonTransparent === null) {
        firstNonTransparent = [data[idx], data[idx + 1], data[idx + 2]]
      } else {
        const colorDiff = Math.abs(data[idx] - firstNonTransparent[0]) +
                         Math.abs(data[idx + 1] - firstNonTransparent[1]) +
                         Math.abs(data[idx + 2] - firstNonTransparent[2])
        if (colorDiff > 30) return false
      }
    }

    return transparentCount > height * 0.95 || transparentCount === 0
  }

  // Find top border
  let detectedCropTop = 0
  for (let y = 0; y < height; y++) {
    if (!isRowEmpty(y)) {
      detectedCropTop = y
      break
    }
  }

  // Find bottom border
  let detectedCropBottom = 0
  for (let y = height - 1; y >= 0; y--) {
    if (!isRowEmpty(y)) {
      detectedCropBottom = height - 1 - y
      break
    }
  }

  // Find left border
  let detectedCropLeft = 0
  for (let x = 0; x < width; x++) {
    if (!isColumnEmpty(x)) {
      detectedCropLeft = x
      break
    }
  }

  // Find right border
  let detectedCropRight = 0
  for (let x = width - 1; x >= 0; x--) {
    if (!isColumnEmpty(x)) {
      detectedCropRight = width - 1 - x
      break
    }
  }

  // Apply the detected crop values
  cropTop.value = detectedCropTop
  cropBottom.value = detectedCropBottom
  cropLeft.value = detectedCropLeft
  cropRight.value = detectedCropRight

  console.log('Auto-detected borders:', {
    top: detectedCropTop,
    bottom: detectedCropBottom,
    left: detectedCropLeft,
    right: detectedCropRight
  })

  // Update canvas
  updateFrameSettings()
}

const detectSpriteGrid = () => {
  if (!uploadedImage.value) return

  // Create a temporary canvas to analyze the cropped portion of the image
  const tempCanvas = document.createElement('canvas')
  const ctx = tempCanvas.getContext('2d')

  // Use cropped dimensions
  const croppedWidth = uploadedImage.value.width - cropLeft.value - cropRight.value
  const croppedHeight = uploadedImage.value.height - cropTop.value - cropBottom.value

  tempCanvas.width = croppedWidth
  tempCanvas.height = croppedHeight

  // Draw only the cropped portion
  ctx.drawImage(
    uploadedImage.value,
    cropLeft.value, cropTop.value,  // Source position
    croppedWidth, croppedHeight,     // Source dimensions
    0, 0,                            // Destination position
    croppedWidth, croppedHeight      // Destination dimensions
  )

  const imageData = ctx.getImageData(0, 0, tempCanvas.width, tempCanvas.height)
  const data = imageData.data
  const width = imageData.width
  const height = imageData.height

  // Function to check if a column is mostly transparent or uniform
  const isColumnEmpty = (x) => {
    let transparentCount = 0
    let firstColor = null
    const tolerance = 10

    for (let y = 0; y < height; y++) {
      const idx = (y * width + x) * 4
      const alpha = data[idx + 3]

      if (alpha < 128) {
        transparentCount++
      } else if (firstColor === null) {
        firstColor = [data[idx], data[idx + 1], data[idx + 2]]
      }
    }

    // Consider column empty if mostly transparent
    return transparentCount > height * 0.9
  }

  // Function to check if a row is mostly transparent or uniform
  const isRowEmpty = (y) => {
    let transparentCount = 0

    for (let x = 0; x < width; x++) {
      const idx = (y * width + x) * 4
      const alpha = data[idx + 3]

      if (alpha < 128) {
        transparentCount++
      }
    }

    return transparentCount > width * 0.9
  }

  // Find vertical gaps (potential column separators)
  const verticalGaps = []
  let inGap = false
  let gapStart = 0

  for (let x = 0; x < width; x++) {
    const empty = isColumnEmpty(x)

    if (empty && !inGap) {
      inGap = true
      gapStart = x
    } else if (!empty && inGap) {
      inGap = false
      if (x - gapStart > 1) { // Only consider gaps wider than 1 pixel
        verticalGaps.push({ start: gapStart, end: x, width: x - gapStart })
      }
    }
  }

  // Find horizontal gaps (potential row separators)
  const horizontalGaps = []
  inGap = false
  gapStart = 0

  for (let y = 0; y < height; y++) {
    const empty = isRowEmpty(y)

    if (empty && !inGap) {
      inGap = true
      gapStart = y
    } else if (!empty && inGap) {
      inGap = false
      if (y - gapStart > 1) {
        horizontalGaps.push({ start: gapStart, end: y, width: y - gapStart })
      }
    }
  }

  // Calculate potential frame dimensions
  let detectedFrameWidth = width
  let detectedFrameHeight = height
  let detectedColumns = 1
  let detectedRows = 1

  // Try to find common divisors if no gaps were found
  const findBestDivisor = (dimension, minSize = 16, maxSize = 256) => {
    const divisors = []
    for (let size = minSize; size <= Math.min(maxSize, dimension); size++) {
      if (dimension % size === 0) {
        divisors.push(size)
      }
    }
    // Prefer sizes that are powers of 2 or common sprite sizes
    const preferredSizes = [16, 32, 48, 64, 96, 128, 192, 256]
    for (const size of preferredSizes) {
      if (divisors.includes(size)) {
        return size
      }
    }
    // Return middle divisor if no preferred size found
    return divisors.length > 0 ? divisors[Math.floor(divisors.length / 2)] : dimension
  }

  // Analyze vertical gaps to find consistent spacing
  let detectedColGap = 0
  if (verticalGaps.length > 1) {
    // Calculate the consistent gap width between frames
    const gapWidths = verticalGaps.map(g => g.width)
    gapWidths.sort((a, b) => a - b)
    detectedColGap = gapWidths[Math.floor(gapWidths.length / 2)] // median gap

    // Calculate frame width including gaps
    if (verticalGaps.length >= 1) {
      detectedFrameWidth = verticalGaps[0].start // First frame width
      detectedColumns = verticalGaps.length + 1
    }
  } else if (verticalGaps.length === 1) {
    // Single gap - probably 2 columns
    detectedFrameWidth = verticalGaps[0].start
    detectedColumns = 2
    detectedColGap = verticalGaps[0].width
  } else {
    // No gaps found, try to find best divisor for width
    detectedFrameWidth = findBestDivisor(width)
    detectedColumns = Math.round(width / detectedFrameWidth)
    detectedColGap = 0
  }

  // Analyze horizontal gaps
  let detectedRowGap = 0
  if (horizontalGaps.length > 1) {
    // Multiple gaps - check if they're consistent
    const gapWidths = horizontalGaps.map(g => g.width)
    const uniqueGapWidths = [...new Set(gapWidths)]

    if (uniqueGapWidths.length === 1) {
      // All gaps are the same size - likely uniform spacing
      detectedRowGap = uniqueGapWidths[0]
      detectedFrameHeight = horizontalGaps[0].start // First frame height
      detectedRows = horizontalGaps.length + 1
    }
  } else if (horizontalGaps.length === 1) {
    // Single gap - probably 2 rows
    detectedFrameHeight = horizontalGaps[0].start
    detectedRows = 2
    detectedRowGap = horizontalGaps[0].width
  } else {
    // No gaps found, try to find best divisor for height
    detectedFrameHeight = findBestDivisor(height)
    detectedRows = Math.round(height / detectedFrameHeight)
    detectedRowGap = 0
  }

  // Update the form values with detected dimensions
  // If gaps were detected, use the exact frame sizes and gaps
  if (detectedColGap > 0 || detectedRowGap > 0) {
    frameWidth.value = detectedFrameWidth
    frameHeight.value = detectedFrameHeight
    columnGap.value = detectedColGap
    rowGap.value = detectedRowGap
  } else {
    // No gaps - ensure frames fill the cropped area
    frameWidth.value = Math.floor(croppedWidth / detectedColumns)
    frameHeight.value = Math.floor(croppedHeight / detectedRows)
    columnGap.value = 0
    rowGap.value = 0
  }
  framesPerRow.value = detectedColumns
  frameCount.value = detectedColumns * detectedRows

  // Redraw canvas with new settings
  drawCanvas()
  drawPreviewFrame()

  console.log('Auto-detected sprite grid:', {
    frameWidth: frameWidth.value,
    frameHeight: frameHeight.value,
    columnGap: columnGap.value,
    rowGap: rowGap.value,
    columns: detectedColumns,
    rows: detectedRows,
    totalFrames: frameCount.value
  })
}

const createSpriteSheetFromGIF = async (arrayBuffer) => {
  const gif = parseGIF(arrayBuffer)
  const frames = decompressFrames(gif, true)

  if (frames.length === 0) return null

  // Get dimensions from first frame
  const firstFrame = frames[0]
  const gifWidth = firstFrame.dims.width
  const gifHeight = firstFrame.dims.height

  // Calculate optimal grid layout
  const numFrames = frames.length
  let cols = Math.ceil(Math.sqrt(numFrames))
  let rows = Math.ceil(numFrames / cols)

  // Create sprite sheet canvas
  const spriteCanvas = document.createElement('canvas')
  spriteCanvas.width = gifWidth * cols
  spriteCanvas.height = gifHeight * rows
  const ctx = spriteCanvas.getContext('2d')

  // Draw each frame to the sprite sheet
  frames.forEach((frame, index) => {
    const col = index % cols
    const row = Math.floor(index / cols)
    const x = col * gifWidth
    const y = row * gifHeight

    // Create ImageData from frame patch
    const imageData = new ImageData(
      new Uint8ClampedArray(frame.patch),
      frame.dims.width,
      frame.dims.height
    )

    // Draw frame to sprite sheet
    const tempCanvas = document.createElement('canvas')
    tempCanvas.width = frame.dims.width
    tempCanvas.height = frame.dims.height
    const tempCtx = tempCanvas.getContext('2d')
    tempCtx.putImageData(imageData, 0, 0)

    ctx.drawImage(tempCanvas, x, y)
  })

  // Convert canvas to image
  return new Promise((resolve) => {
    spriteCanvas.toBlob((blob) => {
      const img = new Image()
      img.onload = () => {
        // Set the frame dimensions based on GIF
        frameWidth.value = gifWidth
        frameHeight.value = gifHeight
        framesPerRow.value = cols
        frameCount.value = numFrames
        columnGap.value = 0
        rowGap.value = 0

        resolve(img)
      }
      img.src = URL.createObjectURL(blob)
    })
  })
}

const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (file && file.type.startsWith('image/')) {
    // Check if it's a GIF
    if (file.type === 'image/gif') {
      // Process as animated GIF
      const reader = new FileReader()
      reader.onload = async (e) => {
        const arrayBuffer = e.target.result
        const spriteSheetImage = await createSpriteSheetFromGIF(arrayBuffer)

        if (spriteSheetImage) {
          uploadedImage.value = spriteSheetImage
          imagePreviewUrl.value = spriteSheetImage.src

          // Reset crop values since this is a generated sprite sheet
          cropTop.value = 0
          cropLeft.value = 0
          cropRight.value = 0
          cropBottom.value = 0

          drawCanvas()
          drawPreviewFrame()
          startAnimation()
        }
      }
      reader.readAsArrayBuffer(file)
    } else {
      // Process as regular image/sprite sheet
      const reader = new FileReader()
      reader.onload = (e) => {
        imagePreviewUrl.value = e.target.result

        const img = new Image()
        img.onload = () => {
          uploadedImage.value = img

          // First detect and crop borders
          detectBorders()

          // Then detect sprite grid within the cropped area
          setTimeout(() => {
            detectSpriteGrid()
          }, 100)

          drawCanvas()
          drawPreviewFrame()
          startAnimation()
        }
        img.src = e.target.result
      }
      reader.readAsDataURL(file)
    }
  }
}

const triggerFileUpload = () => {
  fileInputRef.value.click()
}

const detectFrameContent = (frameIndex) => {
  if (!uploadedImage.value) return null

  const row = Math.floor(frameIndex / framesPerRow.value)
  const col = frameIndex % framesPerRow.value

  const sx = cropLeft.value + (col * (frameWidth.value + columnGap.value))
  const sy = cropTop.value + (row * (frameHeight.value + rowGap.value))

  // Check if frame is within image bounds
  if (sx >= uploadedImage.value.width || sy >= uploadedImage.value.height) {
    return null
  }

  const actualWidth = Math.min(frameWidth.value, uploadedImage.value.width - sx)
  const actualHeight = Math.min(frameHeight.value, uploadedImage.value.height - sy)

  // Create temporary canvas for this frame
  const tempCanvas = document.createElement('canvas')
  const ctx = tempCanvas.getContext('2d')
  tempCanvas.width = actualWidth
  tempCanvas.height = actualHeight

  // Draw the frame
  ctx.drawImage(
    uploadedImage.value,
    sx, sy, actualWidth, actualHeight,
    0, 0, actualWidth, actualHeight
  )

  const imageData = ctx.getImageData(0, 0, actualWidth, actualHeight)
  const data = imageData.data

  let minX = actualWidth
  let minY = actualHeight
  let maxX = 0
  let maxY = 0
  let hasContent = false

  // Find content bounds
  for (let y = 0; y < actualHeight; y++) {
    for (let x = 0; x < actualWidth; x++) {
      const idx = (y * actualWidth + x) * 4
      const alpha = data[idx + 3]

      // Check if pixel is not transparent
      if (alpha > 10) {
        hasContent = true
        minX = Math.min(minX, x)
        minY = Math.min(minY, y)
        maxX = Math.max(maxX, x)
        maxY = Math.max(maxY, y)
      }
    }
  }

  if (!hasContent) {
    return null
  }

  return {
    x: minX,
    y: minY,
    width: maxX - minX + 1,
    height: maxY - minY + 1,
    centerX: (minX + maxX) / 2,
    centerY: (minY + maxY) / 2
  }
}

const calculateFrameCentering = () => {
  if (!uploadedImage.value || !reduceJitter.value) {
    frameBounds.value.clear()
    return
  }

  const newBounds = new Map()
  for (let i = 0; i < frameCount.value; i++) {
    const bounds = detectFrameContent(i)
    if (bounds) {
      newBounds.set(i, bounds)
    }
  }
  frameBounds.value = newBounds
}

const drawPreviewFrame = () => {
  if (!previewCanvasRef.value || !uploadedImage.value) return

  const ctx = previewCanvasRef.value.getContext('2d')
  const canvas = previewCanvasRef.value

  ctx.clearRect(0, 0, canvas.width, canvas.height)

  if (!transparentBackground.value) {
    ctx.fillStyle = backgroundColor.value
    ctx.fillRect(0, 0, canvas.width, canvas.height)
  }

  const row = Math.floor(currentFrame.value / framesPerRow.value)
  const col = currentFrame.value % framesPerRow.value

  // Calculate frame position in the original image coordinates
  // Frames start from the crop position and are spaced by frame width/height plus gaps
  const sx = cropLeft.value + (col * (frameWidth.value + columnGap.value))
  const sy = cropTop.value + (row * (frameHeight.value + rowGap.value))
  const sw = frameWidth.value
  const sh = frameHeight.value

  // Check if the frame is completely outside the image bounds
  if (sx >= uploadedImage.value.width || sy >= uploadedImage.value.height) {
    return // Frame is completely outside the image
  }

  // Clip frame dimensions if they extend beyond the image bounds
  const actualFrameWidth = Math.min(sw, uploadedImage.value.width - sx)
  const actualFrameHeight = Math.min(sh, uploadedImage.value.height - sy)

  const scale = Math.min(
    canvas.width / actualFrameWidth,
    canvas.height / actualFrameHeight
  )

  const dw = actualFrameWidth * scale
  const dh = actualFrameHeight * scale
  const dx = (canvas.width - dw) / 2
  const dy = (canvas.height - dh) / 2

  // Apply jitter reduction (centering) if enabled
  if (reduceJitter.value && frameBounds.value.has(currentFrame.value)) {
    const bounds = frameBounds.value.get(currentFrame.value)

    // Calculate the offset to center the content
    const contentCenterX = bounds.centerX
    const contentCenterY = bounds.centerY
    const frameCenterX = actualFrameWidth / 2
    const frameCenterY = actualFrameHeight / 2

    // Draw only the content bounds, centered in the canvas
    const contentScale = Math.min(
      canvas.width / bounds.width,
      canvas.height / bounds.height
    ) * 0.9

    const drawWidth = bounds.width * contentScale
    const drawHeight = bounds.height * contentScale
    const drawX = (canvas.width - drawWidth) / 2
    const drawY = (canvas.height - drawHeight) / 2

    ctx.drawImage(
      uploadedImage.value,
      sx + bounds.x, sy + bounds.y, bounds.width, bounds.height,
      drawX, drawY, drawWidth, drawHeight
    )
  } else {
    // Normal drawing without jitter reduction
    ctx.drawImage(
      uploadedImage.value,
      sx, sy, actualFrameWidth, actualFrameHeight,
      dx, dy, dw, dh
    )
  }
}

const startAnimation = () => {
  if (!uploadedImage.value || frameCount.value <= 1) return

  stopAnimation()
  isPlaying.value = true

  animationInterval.value = setInterval(() => {
    currentFrame.value = (currentFrame.value + 1) % frameCount.value
    drawPreviewFrame()
  }, frameDelay.value)
}

const stopAnimation = () => {
  if (animationInterval.value) {
    clearInterval(animationInterval.value)
    animationInterval.value = null
  }
  isPlaying.value = false
}

const toggleAnimation = () => {
  if (isPlaying.value) {
    stopAnimation()
  } else {
    startAnimation()
  }
}

const drawCanvas = () => {
  if (!canvasRef.value || !uploadedImage.value) return

  const ctx = canvasRef.value.getContext('2d')
  ctx.clearRect(0, 0, canvasWidth.value, canvasHeight.value)

  if (!transparentBackground.value) {
    ctx.fillStyle = backgroundColor.value
    ctx.fillRect(0, 0, canvasWidth.value, canvasHeight.value)
  }

  // Calculate scale to fit the entire image within the canvas
  const scale = Math.min(
    canvasWidth.value / uploadedImage.value.width,
    canvasHeight.value / uploadedImage.value.height
  ) * 0.9  // 0.9 to leave some padding around the image

  const drawWidth = uploadedImage.value.width * scale
  const drawHeight = uploadedImage.value.height * scale
  const x = (canvasWidth.value - drawWidth) / 2
  const y = (canvasHeight.value - drawHeight) / 2

  // Draw the entire image
  ctx.drawImage(uploadedImage.value, x, y, drawWidth, drawHeight)

  // Draw red overlay for cropped areas
  if (cropTop.value > 0 || cropBottom.value > 0 || cropLeft.value > 0 || cropRight.value > 0) {
    ctx.fillStyle = 'rgba(255, 0, 0, 0.4)'

    // Top cropped area
    if (cropTop.value > 0) {
      ctx.fillRect(x, y, drawWidth, cropTop.value * scale)
    }

    // Bottom cropped area
    if (cropBottom.value > 0) {
      ctx.fillRect(x, y + drawHeight - cropBottom.value * scale, drawWidth, cropBottom.value * scale)
    }

    // Left cropped area
    if (cropLeft.value > 0) {
      ctx.fillRect(x, y, cropLeft.value * scale, drawHeight)
    }

    // Right cropped area
    if (cropRight.value > 0) {
      ctx.fillRect(x + drawWidth - cropRight.value * scale, y, cropRight.value * scale, drawHeight)
    }
  }

  // Calculate the visible (non-cropped) area
  const visibleX = x + cropLeft.value * scale
  const visibleY = y + cropTop.value * scale
  const visibleWidth = (uploadedImage.value.width - cropLeft.value - cropRight.value) * scale
  const visibleHeight = (uploadedImage.value.height - cropTop.value - cropBottom.value) * scale

  // Draw border around the visible area
  ctx.strokeStyle = '#fff'
  ctx.lineWidth = 2
  ctx.strokeRect(visibleX, visibleY, visibleWidth, visibleHeight)

  // Draw frame grid overlay (can extend beyond the visible area into the cropped regions)
  ctx.strokeStyle = '#42b883'
  ctx.lineWidth = 2
  ctx.setLineDash([5, 5])

  const scaledFrameWidth = frameWidth.value * scale
  const scaledFrameHeight = frameHeight.value * scale
  const scaledColumnGap = columnGap.value * scale
  const scaledRowGap = rowGap.value * scale

  for (let i = 0; i < frameCount.value; i++) {
    const row = Math.floor(i / framesPerRow.value)
    const col = i % framesPerRow.value

    // Calculate frame position starting from the crop origin, including gaps
    const frameX = x + (cropLeft.value * scale) + (col * (scaledFrameWidth + scaledColumnGap))
    const frameY = y + (cropTop.value * scale) + (row * (scaledFrameHeight + scaledRowGap))

    // Draw the frame if any part of it is within the image bounds
    if (frameX < x + drawWidth && frameY < y + drawHeight) {
      // Clip the frame to the image bounds
      const clippedX = Math.max(frameX, x)
      const clippedY = Math.max(frameY, y)
      const clippedWidth = Math.min(frameX + scaledFrameWidth, x + drawWidth) - clippedX
      const clippedHeight = Math.min(frameY + scaledFrameHeight, y + drawHeight) - clippedY

      if (clippedWidth > 0 && clippedHeight > 0) {
        ctx.strokeRect(clippedX, clippedY, clippedWidth, clippedHeight)
      }
    }
  }

  ctx.setLineDash([])

  // Draw outer border around the entire image
  ctx.strokeStyle = '#333'
  ctx.lineWidth = 3
  ctx.strokeRect(x - 1, y - 1, drawWidth + 2, drawHeight + 2)
}

const generateGif = async () => {
  if (!uploadedImage.value || isGenerating.value) return

  isGenerating.value = true
  generationProgress.value = 0

  try {
    // Determine output dimensions
    let outputWidth = frameWidth.value
    let outputHeight = frameHeight.value

    // If using jitter reduction, find the maximum content dimensions
    if (reduceJitter.value && frameBounds.value.size > 0) {
      let maxContentWidth = 0
      let maxContentHeight = 0
      for (const bounds of frameBounds.value.values()) {
        maxContentWidth = Math.max(maxContentWidth, bounds.width)
        maxContentHeight = Math.max(maxContentHeight, bounds.height)
      }
      // Use content dimensions with some padding
      outputWidth = Math.ceil(maxContentWidth * 1.1)
      outputHeight = Math.ceil(maxContentHeight * 1.1)
    }

    // Create a GIF encoder
    const gif = new GIF({
      workers: 4,
      quality: 1,  // Lower is better quality (1-30, 1 is best)
      width: outputWidth,
      height: outputHeight,
      transparent: transparentBackground.value ? 0x00000000 : null,
      repeat: loopCount.value === 0 ? 0 : loopCount.value - 1,
      workerScript: '/gif.worker.js',
      dither: false  // Disable dithering for cleaner sprites
    })

    // Create a temporary canvas for each frame
    const tempCanvas = document.createElement('canvas')
    tempCanvas.width = outputWidth
    tempCanvas.height = outputHeight

    const ctx = tempCanvas.getContext('2d', {
      alpha: true,
      willReadFrequently: true
    })

    // Set image smoothing for better quality
    ctx.imageSmoothingEnabled = false  // Disable smoothing for pixel art
    ctx.imageSmoothingQuality = 'high'

    // Add each frame to the GIF
    for (let i = 0; i < frameCount.value; i++) {
      const row = Math.floor(i / framesPerRow.value)
      const col = i % framesPerRow.value

      const sx = cropLeft.value + (col * (frameWidth.value + columnGap.value))
      const sy = cropTop.value + (row * (frameHeight.value + rowGap.value))

      // Skip frames outside image bounds
      if (sx >= uploadedImage.value.width || sy >= uploadedImage.value.height) {
        continue
      }

      const actualFrameWidth = Math.min(frameWidth.value, uploadedImage.value.width - sx)
      const actualFrameHeight = Math.min(frameHeight.value, uploadedImage.value.height - sy)

      // Clear canvas
      ctx.clearRect(0, 0, tempCanvas.width, tempCanvas.height)

      // Fill background if not transparent
      if (!transparentBackground.value) {
        ctx.fillStyle = backgroundColor.value
        ctx.fillRect(0, 0, tempCanvas.width, tempCanvas.height)
      }

      if (reduceJitter.value && frameBounds.value.has(i)) {
        // Draw with jitter reduction - center the content
        const bounds = frameBounds.value.get(i)
        const centerX = outputWidth / 2
        const centerY = outputHeight / 2
        const offsetX = Math.round(centerX - bounds.width / 2)
        const offsetY = Math.round(centerY - bounds.height / 2)

        ctx.drawImage(
          uploadedImage.value,
          sx + bounds.x, sy + bounds.y, bounds.width, bounds.height,
          offsetX, offsetY, bounds.width, bounds.height
        )
      } else {
        // Draw normally - preserve original size
        const drawX = Math.round((outputWidth - actualFrameWidth) / 2)
        const drawY = Math.round((outputHeight - actualFrameHeight) / 2)

        ctx.drawImage(
          uploadedImage.value,
          sx, sy, actualFrameWidth, actualFrameHeight,
          drawX, drawY, actualFrameWidth, actualFrameHeight
        )
      }

      // Add frame to GIF
      gif.addFrame(ctx, { copy: true, delay: frameDelay.value })

      generationProgress.value = ((i + 1) / frameCount.value) * 100
    }

    // Set up progress tracking
    gif.on('progress', (progress) => {
      generationProgress.value = Math.round(progress * 100)
    })

    // Render the GIF
    gif.on('finished', (blob) => {
      // Create download link
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = 'sprite-animation.gif'
      document.body.appendChild(a)
      a.click()
      document.body.removeChild(a)
      URL.revokeObjectURL(url)

      isGenerating.value = false
      generationProgress.value = 0
    })

    gif.render()

  } catch (error) {
    console.error('Error generating GIF:', error)
    isGenerating.value = false
    generationProgress.value = 0
  }
}

const resetSettings = () => {
  frameWidth.value = 64
  frameHeight.value = 64
  frameCount.value = 12
  framesPerRow.value = 6
  columnGap.value = 0
  rowGap.value = 0
  frameDelay.value = 100
  loopCount.value = 0
  backgroundColor.value = '#FFFFFF'
  transparentBackground.value = false
  cropTop.value = 0
  cropBottom.value = 0
  cropLeft.value = 0
  cropRight.value = 0
  currentFrame.value = 0
  stopAnimation()
  drawCanvas()
  drawPreviewFrame()
}

const updateAnimationSpeed = () => {
  if (isPlaying.value) {
    stopAnimation()
    startAnimation()
  }
}

const handleReduceJitterChange = () => {
  if (reduceJitter.value) {
    calculateFrameCentering()
  } else {
    frameBounds.value.clear()
  }
  drawPreviewFrame()
}

const updateFrameSettings = () => {
  currentFrame.value = 0

  // Don't automatically adjust frame count or frames per row
  // Let the user manually control these values
  // The grid can extend beyond the cropped area if needed

  // Recalculate frame bounds if jitter reduction is enabled
  if (reduceJitter.value) {
    calculateFrameCentering()
  }

  drawCanvas()
  drawPreviewFrame()
  if (isPlaying.value) {
    stopAnimation()
    startAnimation()
  }
}

onMounted(() => {
  if (canvasRef.value) {
    canvasRef.value.width = canvasRef.value.offsetWidth
    canvasRef.value.height = canvasRef.value.offsetHeight
  }

  if (previewCanvasRef.value) {
    previewCanvasRef.value.width = 200
    previewCanvasRef.value.height = 200
  }

  const resizeCanvas = () => {
    if (canvasRef.value) {
      canvasRef.value.width = canvasRef.value.offsetWidth
      canvasRef.value.height = canvasRef.value.offsetHeight
      drawCanvas()
    }
  }

  window.addEventListener('resize', resizeCanvas)

  // Store resize handler for cleanup
  window.resizeHandler = resizeCanvas
})

onUnmounted(() => {
  stopAnimation()
  if (window.resizeHandler) {
    window.removeEventListener('resize', window.resizeHandler)
  }
})
</script>

<template>
  <div class="app-container">
    <div class="canvas-section">
      <canvas ref="canvasRef" class="main-canvas"></canvas>
    </div>

    <div class="controls-section">
      <div class="preview-section">
        <h3>Animation Preview</h3>
        <div class="animation-preview">
          <canvas v-if="uploadedImage" ref="previewCanvasRef" class="preview-canvas"></canvas>
          <div v-else class="upload-placeholder" @click="triggerFileUpload">
            <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="3" width="18" height="18" rx="2" ry="2"/>
              <circle cx="8.5" cy="8.5" r="1.5"/>
              <polyline points="21 15 16 10 5 21"/>
            </svg>
            <p>Click to upload sprite sheet</p>
          </div>
        </div>
        <div v-if="uploadedImage && frameCount > 1" class="animation-controls">
          <button @click="toggleAnimation" class="btn btn-play">
            <svg v-if="!isPlaying" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
              <polygon points="5 3 19 12 5 21 5 3"/>
            </svg>
            <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
              <rect x="6" y="4" width="4" height="16"/>
              <rect x="14" y="4" width="4" height="16"/>
            </svg>
            {{ isPlaying ? 'Pause' : 'Play' }}
          </button>
          <span class="frame-counter">Frame: {{ currentFrame + 1 }} / {{ frameCount }}</span>
        </div>
        <input
          ref="fileInputRef"
          type="file"
          @change="handleFileUpload"
          accept="image/*"
          style="display: none"
        />
      </div>

      <div class="form-section">
        <h3>Conversion Settings</h3>

        <div v-if="uploadedImage" class="auto-detect-section">
          <button @click="detectSpriteGrid" class="btn btn-auto-detect">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="3"/>
              <path d="M12 1v6m0 6v6m11-7h-6m-6 0H1"/>
            </svg>
            Auto-Detect Grid
          </button>
        </div>

        <div class="form-group">
          <label for="frame-width">Frame Width (px)</label>
          <input
            id="frame-width"
            type="number"
            v-model.number="frameWidth"
            @input="updateFrameSettings"
            min="1"
          />
        </div>

        <div class="form-group">
          <label for="frame-height">Frame Height (px)</label>
          <input
            id="frame-height"
            type="number"
            v-model.number="frameHeight"
            @input="updateFrameSettings"
            min="1"
          />
        </div>

        <div class="form-group">
          <label for="frame-count">Number of Frames</label>
          <input
            id="frame-count"
            type="number"
            v-model.number="frameCount"
            @input="updateFrameSettings"
            min="1"
          />
        </div>

        <div class="form-group">
          <label for="frames-per-row">Frames Per Row</label>
          <input
            id="frames-per-row"
            type="number"
            v-model.number="framesPerRow"
            @input="updateFrameSettings"
            min="1"
          />
        </div>

        <div class="form-group">
          <label for="column-gap">Column Gap (px)</label>
          <input
            id="column-gap"
            type="number"
            v-model.number="columnGap"
            @input="updateFrameSettings"
            min="0"
          />
        </div>

        <div class="form-group">
          <label for="row-gap">Row Gap (px)</label>
          <input
            id="row-gap"
            type="number"
            v-model.number="rowGap"
            @input="updateFrameSettings"
            min="0"
          />
        </div>

        <div class="form-group">
          <label for="frame-delay">Frame Delay (ms)</label>
          <input
            id="frame-delay"
            type="number"
            v-model.number="frameDelay"
            @input="updateAnimationSpeed"
            min="10"
            step="10"
          />
        </div>

        <div class="form-group">
          <label for="loop-count">Loop Count (0 = infinite)</label>
          <input
            id="loop-count"
            type="number"
            v-model.number="loopCount"
            min="0"
          />
        </div>

        <h3 style="margin-top: 20px; margin-bottom: 15px;">Sprite Sheet Cropping</h3>

        <div class="box-model-cropping">
          <div class="crop-label">sheet crop</div>

          <div class="crop-top">
            <input
              type="number"
              v-model.number="cropTop"
              @input="updateFrameSettings"
              min="0"
              :max="uploadedImage ? uploadedImage.height - 1 : 0"
              class="crop-input"
            />
          </div>

          <div class="crop-middle">
            <div class="crop-left">
              <input
                type="number"
                v-model.number="cropLeft"
                @input="updateFrameSettings"
                min="0"
                :max="uploadedImage ? uploadedImage.width - 1 : 0"
                class="crop-input"
              />
            </div>

            <div class="crop-center">
              <div class="frame-size-display">
                {{ uploadedImage ? uploadedImage.width - cropLeft - cropRight : 0 }} × {{ uploadedImage ? uploadedImage.height - cropTop - cropBottom : 0 }}
              </div>
            </div>

            <div class="crop-right">
              <input
                type="number"
                v-model.number="cropRight"
                @input="updateFrameSettings"
                min="0"
                :max="uploadedImage ? uploadedImage.width - 1 : 0"
                class="crop-input"
              />
            </div>
          </div>

          <div class="crop-bottom">
            <input
              type="number"
              v-model.number="cropBottom"
              @input="updateFrameSettings"
              min="0"
              :max="uploadedImage ? uploadedImage.height - 1 : 0"
              class="crop-input"
            />
          </div>
        </div>

        <div class="form-group">
          <label for="bg-color">Background Color</label>
          <div class="color-input-group">
            <input
              id="bg-color"
              type="color"
              v-model="backgroundColor"
              @input="drawCanvas"
              :disabled="transparentBackground"
            />
            <input
              type="text"
              v-model="backgroundColor"
              @input="drawCanvas"
              :disabled="transparentBackground"
              placeholder="#FFFFFF"
            />
          </div>
        </div>

        <div class="form-group checkbox-group">
          <input
            id="transparent-bg"
            type="checkbox"
            v-model="transparentBackground"
            @change="drawCanvas"
          />
          <label for="transparent-bg">Transparent Background</label>
        </div>

        <div class="form-group checkbox-group">
          <input
            id="reduce-jitter"
            type="checkbox"
            v-model="reduceJitter"
            @change="handleReduceJitterChange"
          />
          <label for="reduce-jitter">Reduce Jitter (Center Frames)</label>
        </div>

        <div class="button-group">
          <button @click="generateGif" class="btn btn-primary" :disabled="!uploadedImage || isGenerating">
            <span v-if="!isGenerating">Generate & Download GIF</span>
            <span v-else>Generating... {{ Math.round(generationProgress) }}%</span>
          </button>
          <button @click="resetSettings" class="btn btn-secondary" :disabled="isGenerating">
            Reset Settings
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  overflow: hidden;
}
</style>

<style scoped>
.app-container {
  display: flex;
  height: 100vh;
  width: 100vw;
  background: #1a1a1a;
}

.canvas-section {
  flex: 4;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #2c2c2c;
  padding: 20px;
}

.main-canvas {
  width: 100%;
  height: 100%;
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.controls-section {
  flex: 1;
  background: #f5f5f5;
  padding: 20px;
  overflow-y: auto;
  border-left: 1px solid #ddd;
}

.preview-section {
  margin-bottom: 30px;
}

.preview-section h3,
.form-section h3 {
  font-size: 1.1rem;
  font-weight: 600;
  margin-bottom: 15px;
  color: #333;
}

.animation-preview {
  width: 100%;
  height: 200px;
  border: 2px solid #ddd;
  border-radius: 8px;
  overflow: hidden;
  transition: border-color 0.3s;
  background: white;
  display: flex;
  align-items: center;
  justify-content: center;
}

.preview-canvas {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.animation-controls {
  margin-top: 10px;
  display: flex;
  align-items: center;
  gap: 15px;
}

.btn-play {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 6px 12px;
  background: #42b883;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background-color 0.3s;
}

.btn-play:hover {
  background: #35a372;
}

.btn-play svg {
  width: 16px;
  height: 16px;
}

.frame-counter {
  font-size: 0.85rem;
  color: #666;
}

.upload-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #999;
  cursor: pointer;
  width: 100%;
}

.upload-placeholder:hover {
  background: #f9f9f9;
}

.upload-placeholder svg {
  color: #ccc;
  margin-bottom: 10px;
}

.upload-placeholder p {
  font-size: 0.9rem;
}

.form-section {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.form-group label {
  font-size: 0.9rem;
  font-weight: 500;
  color: #555;
}

.form-group input[type="number"],
.form-group input[type="text"] {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 0.95rem;
  transition: border-color 0.3s;
}

.form-group input[type="number"]:focus,
.form-group input[type="text"]:focus {
  outline: none;
  border-color: #42b883;
}

.form-group input:disabled {
  background: #f0f0f0;
  color: #999;
  cursor: not-allowed;
}

.color-input-group {
  display: flex;
  gap: 10px;
}

.color-input-group input[type="color"] {
  width: 50px;
  height: 36px;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
}

.color-input-group input[type="color"]:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

.color-input-group input[type="text"] {
  flex: 1;
}

.box-model-cropping {
  position: relative;
  width: 100%;
  max-width: 280px;
  margin: 0 auto 20px;
  border: 2px dashed #999;
  background: #f5deb3;
  padding: 8px;
}

.crop-label {
  position: absolute;
  top: 2px;
  left: 8px;
  font-size: 0.8rem;
  color: #666;
  font-weight: 500;
  background: #f5deb3;
  padding: 0 4px;
}

.crop-top,
.crop-bottom {
  text-align: center;
  padding: 4px;
}

.crop-middle {
  display: flex;
  align-items: center;
  justify-content: space-between;
  min-height: 80px;
}

.crop-left,
.crop-right {
  flex: 0 0 auto;
  display: flex;
  align-items: center;
  padding: 0 4px;
}

.crop-center {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #90ee90;
  border: 2px dashed #666;
  min-height: 60px;
  margin: 0 8px;
  padding: 10px;
}

.frame-size-display {
  font-size: 1.1rem;
  font-weight: 600;
  color: #333;
  text-align: center;
  background: #e6f3ff;
  padding: 8px 12px;
  border-radius: 4px;
  white-space: nowrap;
}

.crop-input {
  width: 50px;
  padding: 4px;
  text-align: center;
  border: 1px solid #ccc;
  border-radius: 3px;
  font-size: 0.9rem;
  background: white;
}

.crop-input:focus {
  outline: none;
  border-color: #42b883;
  box-shadow: 0 0 0 2px rgba(66, 184, 131, 0.2);
}

.checkbox-group {
  flex-direction: row;
  align-items: center;
  gap: 10px;
}

.checkbox-group input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.button-group {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.btn {
  flex: 1;
  padding: 10px 16px;
  font-size: 0.95rem;
  font-weight: 500;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s;
}

.btn-primary {
  background: #42b883;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #35a372;
}

.btn-primary:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.btn-secondary {
  background: #6c757d;
  color: white;
}

.btn-secondary:hover {
  background: #5a6268;
}

.auto-detect-section {
  margin-bottom: 20px;
}

.btn-auto-detect {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 10px 16px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.95rem;
  font-weight: 500;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
}

.btn-auto-detect:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.btn-auto-detect svg {
  width: 16px;
  height: 16px;
}
</style>