<script lang="ts">
    // import { spring } from "svelte/motion";

    export let width = 640, height = 640
    const DAMP = 0.9
    const HIT_VELOCITY = 0.1
    const VELOCITY_CUTOFF = 0.01

    import { Canvas, Layer, t } from "svelte-canvas"

    let mouse = { show: false, x: 0, y: 0 }
    let ball = { moving: false, x: width / 2, y: height / 2, vx: 0, vy: 0 }

    $: render = ({ context, width, height }) => {
        animate()
        //if ($t % 1000 < 30) console.log(mouse.x, mouse.y, $t)
        context.fillStyle = `hsl(${$t / 40}, 100%, 50%)`
        context.beginPath()
        context.arc(ball.x, ball.y, 100, 0, Math.PI * 2)
        context.closePath()
        context.fill()
        if (mouse.show) {
            context.beginPath()
            context.moveTo(ball.x, ball.y)
            context.lineTo(mouse.x, mouse.y)
            context.stroke()
        }
    }

    function animate() {
        if (ball.moving) {
            ball.x += ball.vx
            ball.y += ball.vy
            ball.vx *= DAMP
            ball.vy *= DAMP
            const v2 = ball.vx * ball.vx + ball.vy * ball.vy
            if (v2 < VELOCITY_CUTOFF) {
                ball = {
                    ...ball,
                    moving: false,
                    vx: 0,
                    vy: 0,
                }
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
</script>

<Canvas
    {width} {height}
    on:mouseleave={() => { mouse = { show: false, x: 0, y: 0 } }}
    on:mousemove={aim}
    on:click={shoot}
>
    <Layer {render} />
</Canvas>