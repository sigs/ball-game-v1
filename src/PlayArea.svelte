<script lang="ts">
    export let width = 640, height = 640

    // import { spring } from "svelte/motion";
    import { Canvas, Layer, t } from "svelte-canvas"

    // TODO: move to map properties
    const DAMP = 0.9
    const HIT_VELOCITY = 0.1
    const VELOCITY_CUTOFF = 0.01
    const BORDER_BUMP_DAMP = 0.8

    let mouse = { show: false, x: 0, y: 0 }
    let ball = { moving: false, r: 20, x: width / 2, y: height / 2, vx: 0, vy: 0 }
    let map = {
        width: 10,
        height: 10,
        tiles: [],
    }

    $: render = ({ context, width, height }) => {
        update() // TODO: update independent of render?
        //if ($t % 1000 < 30) console.log(mouse.x, mouse.y, $t)

        context.fillStyle = "#26C329"
        context.fillRect(0, 0, width, height)

        if (mouse.show && !ball.moving) {
            context.beginPath()
            context.lineWidth = 2
            context.moveTo(ball.x, ball.y)
            context.lineTo(mouse.x, mouse.y)
            context.stroke()
        }
        const gradient = context.createRadialGradient(ball.x-7, ball.y-5, 0, ball.x-7, ball.y, ball.r * 1.5)
        gradient.addColorStop(0, "white");
        gradient.addColorStop(.7, "blue");
        gradient.addColorStop(1, "darkblue");
        context.fillStyle = gradient
        context.beginPath()
        context.arc(ball.x, ball.y, ball.r, 0, Math.PI * 2)
        context.closePath()
        context.fill()
    }

    function update() {
        if (ball.moving) {
            updateBall()
        }
    }

    function updateBall() {
        ball.x += ball.vx
        ball.y += ball.vy
        ball.vx *= DAMP
        ball.vy *= DAMP
        const v2 = ball.vx * ball.vx + ball.vy * ball.vy

        if (ball.x < ball.r) {
            ball.x = 2 * ball.r - ball.x
            ball.vx *= -BORDER_BUMP_DAMP
        }
        if (ball.x > width - ball.r) {
            ball.x = 2 * (width - ball.r) - ball.x
            ball.vx *= -BORDER_BUMP_DAMP
        }
        if (ball.y < ball.r) {
            ball.y = 2 * ball.r - ball.y
            ball.vy *= -BORDER_BUMP_DAMP
        }
        if (ball.y > height - ball.r) {
            ball.y = 2 * (height - ball.r) - ball.y
            ball.vy *= -BORDER_BUMP_DAMP
        }

        if (v2 < VELOCITY_CUTOFF) {
            ball = {
                ...ball,
                moving: false,
                vx: 0,
                vy: 0,
            }
        }
    }

    function aim(point: MouseEvent) {//: {clientX: number, clientY: number}) {
        mouse = { show: true, x: point.offsetX, y: point.offsetY }
    }

    function shoot(point: MouseEvent) {
        if (ball.moving) { return }
        ball = {
            ...ball,
            moving: true,
            vx: (point.offsetX - ball.x) * HIT_VELOCITY,
            vy: (point.offsetY - ball.y) * HIT_VELOCITY,
        }
    }
    function shootReverse(point: MouseEvent) {
        if (ball.moving) { return }
        ball = {
            ...ball,
            moving: true,
            vx: -(point.offsetX - ball.x) * HIT_VELOCITY,
            vy: -(point.offsetY - ball.y) * HIT_VELOCITY,
        }
    }
</script>

<Canvas
    {width} {height}
    on:mouseleave={() => { mouse = { show: false, x: 0, y: 0 } }}
    on:mousemove={aim}
    on:click={shoot}
    on:contextmenu={(e) => { e.preventDefault(); shootReverse(e); return false }}
>
    <Layer {render} />
</Canvas>