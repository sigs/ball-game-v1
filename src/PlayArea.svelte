<script lang="ts">
    export let width = 640, height = 640

    // import { spring } from "svelte/motion";
    import { Canvas, Layer, t } from "svelte-canvas"

    // TODO: move to map properties
    const DAMP = 0.9
    const HIT_VELOCITY = 0.1
    const VELOCITY_CUTOFF = 0.0003
    const BORDER_BUMP_DAMP = 0.8

    const BORDER_WIDTH_PX = 5

    let strikeSound, waterSound, winSound

    const terrainImages = new Image()
    terrainImages.src = "res/track_elements.gif"
    const objectImages = new Image()
    objectImages.src = "res/track_objects.png"
    let mapImage
    enum Tile {
        GRASS, SAND, MUD, ICE,
        GRASS_U, GRASS_RU, GRASS_R, GRASS_RD,
        GRASS_D, GRASS_LD, GRASS_L, GRASS_LU,
        WATER, YELLOW, GRASS_WATER, GRASS_YELLOW, // what is that yellow thing?
        WALL, BEIGE, TAN, WALL_DARK,
        WALL_U, WALL_R, WALL_D, WALL_L,
    }
    enum Object {
        NONE, HOLE,
    }

    // units are "map units" so that resizing the canvas won't mess things up
    let map = {
        width: 20,
        height: 20,
        start_x: 10,
        start_y: 10,
        tiles: [[Tile.GRASS]],
        objects: [[]]
    }
    loadMap()
    let mouse = { show: false, x: 0, y: 0, sx: 0, sy: 0 }
    let ball = { moving: false, r: 0.5, x: map.start_x, y: map.start_y, vx: 0, vy: 0 }

    let tilesize = 1
    let padding_x = 0, padding_y = 0
    function mapToScreen(x, y) {
        return [
            x * tilesize + padding_x,
            y * tilesize + padding_y,
        ]
    }
    function screenToMap(x, y) {
        return [
            (x - padding_x) / tilesize,
            (y - padding_y) / tilesize,
        ]
    }

    $: render = ({ context, width, height }) => {
        update() // TODO: update independent of render?
        //if ($t % 1000 < 30) console.log(mouse.x, mouse.y, $t)

        if (!mapImage || mapImage.width !== width || mapImage.height !== height) {
            console.log("redraw")
            if (terrainImages.width < 1) {
                return
            }
            drawMap(context, width, height)
        }
        context.putImageData(mapImage, 0, 0)
        //console.log(mapImage.width, mapImage.height, width, height)

        const [ball_sx, ball_sy] = mapToScreen(ball.x, ball.y)
        const ball_sr = ball.r * tilesize

        if (mouse.show && !ball.moving) {
            context.beginPath()
            context.lineWidth = 2
            context.moveTo(ball_sx, ball_sy)
            context.lineTo(mouse.sx, mouse.sy)
            context.stroke()
        }
        const gradient = context.createRadialGradient(ball_sx-7, ball_sy-5, 0, ball_sx-7, ball_sy, ball_sr * 1.5)
        gradient.addColorStop(0, "white");
        gradient.addColorStop(.7, "blue");
        gradient.addColorStop(1, "darkblue");
        context.fillStyle = gradient
        context.beginPath()
        context.arc(ball_sx, ball_sy, ball_sr, 0, Math.PI * 2)
        context.closePath()
        context.fill()
    }

    function drawMap(context, canvasWidth, canvasHeight) {

        // TODO: instead of padding, maybe resize canvas so that it has correct ratio for map?
        tilesize = Math.floor(Math.min((canvasWidth - BORDER_WIDTH_PX) / map.width, (canvasHeight - BORDER_WIDTH_PX) / map.height))
        const sw = tilesize * map.width
        const sh = tilesize * map.height
        padding_x = (canvasWidth - sw) / 2
        padding_y = (canvasHeight - sh) / 2

        context.fillStyle = "#26C329"
        context.fillRect(0, 0, canvasWidth, canvasHeight)

        context.lineWidth = BORDER_WIDTH_PX
        context.setLineDash([5, 3])
        context.beginPath()
        context.rect(padding_x-BORDER_WIDTH_PX/2+1, padding_y-BORDER_WIDTH_PX/2+1, sw+BORDER_WIDTH_PX, sh+BORDER_WIDTH_PX)
        context.stroke()
        context.setLineDash([])

        for (let y = 0; y < map.height; y++) {
            for (let x = 0; x < map.width; x++) {
                const tx = map.tiles[y][x] % 4
                const ty = Math.floor(map.tiles[y][x] / 4)
                context.drawImage(
                    terrainImages,
                    tx*16+2, ty*16+2, 13, 13,
                    padding_x + tilesize * x, padding_y + tilesize * y, tilesize, tilesize
                )
                const ox = map.objects[y][x] % 9
                const oy = Math.floor(map.objects[y][x] / 9)
                context.drawImage(
                    objectImages,
                    ox*16+1, oy*16+1, 15, 15,
                    padding_x + tilesize * x, padding_y + tilesize * y, tilesize, tilesize
                )
            }
        }

        mapImage = context.getImageData(0, 0, canvasWidth, canvasHeight)
    }

    function loadMap() {
        map = {
            width: 20,
            height: 20,
            start_x: 10,
            start_y: 10,
            tiles: Array(map.height).fill(0).map(y => Array(map.width).fill(Tile.GRASS)),
            objects: Array(map.height).fill(0).map(y => Array(map.width).fill(Object.NONE)),
        }
        for (let y = 0; y < map.height; y++) {
            for (let x = 0; x < map.width; x++) {
                const xx = x - map.width/2 + 0.1
                const yy = y - map.height/2 + 0.4
                const a = Math.atan2(yy, xx) / Math.PI + 1
                const r = Math.sqrt(xx*xx + yy*yy)
                map.tiles[y][x] = (r/2 + a) % 2 < 1 || r < 1 ? Tile.GRASS
                    : (a * Math.floor(r/2 + a)) % 1.0 < 0.2 ? Tile.SAND : Tile.WATER
            }
        }
        map.objects[19][10] = Object.HOLE
    }

    function update() {
        if (ball.moving) {
            updateBall()
        }
    }

    function updateBall() {
        const terrain = map.tiles[Math.floor(ball.y)][Math.floor(ball.x)]
        const object = map.objects[Math.floor(ball.y)][Math.floor(ball.x)]

        if (object === Object.HOLE) {
            ball = {
                //...ball,
                r: ball.r,
                moving: false,
                x: map.start_x,
                y: map.start_y,
                vx: 0,
                vy: 0,
            }
            winSound.play()
            return
        }
        if (terrain === Tile.WATER) {
            ball = {
                //...ball,
                r: ball.r,
                moving: false,
                x: map.start_x,
                y: map.start_y,
                vx: 0,
                vy: 0,
            }
            waterSound.play()
            return
        }
        const terrain_damp = terrain === Tile.SAND ? 0.8 : 1

        ball.x += ball.vx
        ball.y += ball.vy
        ball.vx *= DAMP * terrain_damp
        ball.vy *= DAMP * terrain_damp
        const v2 = ball.vx * ball.vx + ball.vy * ball.vy

        if (ball.x < ball.r) {
            ball.x = 2 * ball.r - ball.x
            ball.vx *= -BORDER_BUMP_DAMP
        }
        if (ball.x > map.width - ball.r) {
            ball.x = 2 * (map.width - ball.r) - ball.x
            ball.vx *= -BORDER_BUMP_DAMP
        }
        if (ball.y < ball.r) {
            ball.y = 2 * ball.r - ball.y
            ball.vy *= -BORDER_BUMP_DAMP
        }
        if (ball.y > map.height - ball.r) {
            ball.y = 2 * (map.height - ball.r) - ball.y
            ball.vy *= -BORDER_BUMP_DAMP
        }

        if (v2 < VELOCITY_CUTOFF) {
            ball = {
                //...ball,
                r: ball.r,
                x: ball.x,
                y: ball.y,
                moving: false,
                vx: 0,
                vy: 0,
            }
        }
    }

    function aim(point: MouseEvent) {
        const sx = point.offsetX
        const sy = point.offsetY
        const [x, y] = screenToMap(sx, sy)
        mouse = { show: true, sx, sy, x, y }
    }

    function shoot(point: MouseEvent) {
        if (ball.moving) { return }
        strikeSound.play()
        const [px, py] = screenToMap(point.offsetX, point.offsetY)
        ball = {
            //...ball,
            r: ball.r,
            x: ball.x,
            y: ball.y,
            moving: true,
            vx: (px - ball.x) * HIT_VELOCITY,
            vy: (py - ball.y) * HIT_VELOCITY,
        }
    }
    function shootReverse(point: MouseEvent) {
        if (ball.moving) { return }
        strikeSound.play()
        const [px, py] = screenToMap(point.offsetX, point.offsetY)
        ball = {
            //...ball,
            r: ball.r,
            x: ball.x,
            y: ball.y,
            moving: true,
            vx: -(px - ball.x) * HIT_VELOCITY,
            vy: -(py - ball.y) * HIT_VELOCITY,
        }
    }
</script>

<!-- svelte-ignore a11y-media-has-caption -->
<audio bind:this={strikeSound} src="res/player_strike.wav"/>
<!-- svelte-ignore a11y-media-has-caption -->
<audio bind:this={waterSound} src="res/ball_water.wav"/>
<!-- svelte-ignore a11y-media-has-caption -->
<audio bind:this={winSound} src="res/game_draw.wav"/>
<Canvas
    {width} {height}
    on:mouseleave={() => { mouse = { show: false, x: 0, y: 0, sx: 0, sy: 0 } }}
    on:mousemove={aim}
    on:click={shoot}
    on:contextmenu={(e) => { e.preventDefault(); shootReverse(e); return false }}
>
    <Layer {render} />
</Canvas>