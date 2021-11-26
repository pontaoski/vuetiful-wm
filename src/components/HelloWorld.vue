<script setup lang="ts">
import { reactive, ref, Ref } from 'vue'
import { animationFrames, animationFrameScheduler, fromEvent, iif, interval, merge, Observable, of, OperatorFunction, timer } from 'rxjs';
import { combineLatest, endWith, last, map, mergeMap, repeat, switchMap, takeUntil, takeWhile, throttle, throttleTime, withLatestFrom } from 'rxjs/operators';

function update<A>(a: A, b: Partial<A>): A {
  let sin: any = {}
  for (let key in a) {
    sin[key] = a[key]
  }
  for (let key in b) {
    sin[key] = b[key]
  }
  return sin as A
}

function lerp(v0: number, v1: number, t: number) {
    return v0*(1-t)+v1*t
}

function lerpR(r1: Rectangle, r2: Rectangle, t: number): Rectangle {
  return new Rectangle( r1.n,
    lerp(r1.x, r2.x, t),
    lerp(r1.y, r2.y, t),
    lerp(r1.w, r2.w, t),
    lerp(r1.h, r2.h, t))
}

class Rectangle {
  x: number
  y: number
  w: number
  h: number
  n: number

  constructor(n: number, x: number, y: number, w: number, h: number) {
    this.n = n
    this.x = x
    this.y = y
    this.w = w
    this.h = h
  }
}

function make(majProp: keyof Rectangle, minProp: keyof Rectangle): (spacing: number, rects: Rectangle[]) => Rectangle[] {
  return (spacing, rects) => {
    let accumulator = 0

    return rects.map(
      (rect) => {
        const yes = update(rect, {[minProp]: 0, [majProp]: accumulator})
        accumulator += rect[majProp] + spacing
        return yes
      }
    )
  }
}

const chunk = <A extends any>(arr: A[], size: number): A[][] =>
  arr
    .reduce((acc: A[][], _, i) =>
      (i % size)
        ? acc
        : [...acc, arr.slice(i, i + size)]
    , [])

const row =
  make("x", "y")

const column =
  make("y", "x")


function transpose<A>(arr: A[][]): A[][] {
  return arr[0]
    .map(
      (x, i) =>
        arr.map(x => x[i]).filter(it => it !== undefined)
    )
}

function gridByRows(rows: number, colSpacing: number, rowSpacing: number): (rects: Rectangle[]) => Rectangle[] {
  return (rects: Rectangle[]): Rectangle[] => {
    const ncols = Math.ceil(rects.length / rows)
    const nrects = rects.slice()
    const cols = []

    // group into a bunch of columns
    for (let i = 0; i < ncols; i++) {
      const col = []
      for (let k = i*rows; k < Math.min(i*rows + rows, rects.length); k++) {
        col.push(nrects[k])
      }
      cols.push(col)
    }

    let xAccumulator = 0
    let yAccumulator = 0

    const max = (a: number, b: number) => Math.max(a, b)

    // set the x positions
    let upd = cols.map((col, idx) => {
      let maxWidth = col.map(item => item.w).reduce(max)
      const nuevo = col.map(item => update(item, {x: xAccumulator}))
      xAccumulator += maxWidth + colSpacing
      return nuevo
    })
    upd = transpose(upd).map(row => {
      let maxHeight = row.map(item => item.h).reduce(max)
      const nuevo = row.map(item => update(item, {y: yAccumulator}))
      yAccumulator += maxHeight + rowSpacing
      return nuevo
    })

    return upd.flat()
  }
}

const src =
  [ new Rectangle(1, 100, 100, 300, 200)
  , new Rectangle(2, 600, 100, 300, 200)
  , new Rectangle(3, 50, 300, 300, 200)
  , new Rectangle(4, 500, 200, 300, 200)
  , new Rectangle(5, 500, 500, 300, 200)
  , new Rectangle(6, 200, 400, 300, 200)
  ]

const items: Ref<Rectangle[]> = ref(src)

const fn = gridByRows(2, 10, 10)

const dst = fn(src).sort((a, b) => a.n - b.n)
const zipped: [Rectangle, Rectangle][] = src.map((e, i) => [e, dst[i]])

const animationFrames$ = animationFrames()

function tween(start: number, end: number, duration: number): Observable<number> {
  const diff = end - start;
  return animationFrames$.pipe(
    // Figure out what percentage of time has passed
    map(({elapsed}) => elapsed / duration),
    // Take the vector while less than 100%
    takeWhile(v => v < 1),
    // Finish with 100%
    endWith(1),
    // Calculate the distance traveled between start and end
    map(v => v * diff + start)
  );
}

function easeInOutQuad(x: number): number {
  return x < 0.5 ? 2 * x * x : 1 - Math.pow(-2 * x + 2, 2) / 2;
}

// setTimeout(() => {
//   tween(0, 1, 500)
//     .pipe(
//       map(easeInOutQuad),
//       map(percent => zipped.map(l => lerpR(l[0], l[1], percent)))
//     )
//     .subscribe(rects => items.value = rects)
// }, 1000)

function latestWhile<T>(source: Observable<boolean>): (obs: Observable<T>) => Observable<T> {
  return target => {
    return new Observable(sub => {
      let current: boolean | undefined
      source.subscribe(val => current = val)
      target.subscribe(wert => { if (current) sub.next(wert) })
    })
  }
}

interface Point {
  x: number
  y: number
}

const move$ = fromEvent(document, 'mousemove') as Observable<MouseEvent>
const down$ = fromEvent(document, 'mousedown') as Observable<MouseEvent>
const up$ = fromEvent(document, 'mouseup') as Observable<MouseEvent>
const isDown$ = merge(
  down$.pipe(map(_ => true)),
  up$.pipe(map(_ => false))
)
const pos$: Observable<Point> = move$.pipe(map(ev => {
  return {x: ev.clientX, y: ev.clientY}
}))
const lastPos$ = pos$.pipe(latestWhile(isDown$))

const moved$ = lastPos$.pipe(
  map(ev => ev.y / document.documentElement.clientHeight),
  map(ev => 1 - ev),
  map(easeInOutQuad),
  map(percent => zipped.map(l => lerpR(l[0], l[1], percent)))
).subscribe(rects => items.value = rects)

defineProps<{ msg: string }>()

</script>

<template>
  <div id="bg">
    <div
      class="window"
      v-for="(item, index) in items"
      :key="index"
      :style="{
        top: `${item.y}px`,
        left: `${item.x}px`,
        width: `${item.w}px`,
        height: `${item.h}px`,
        'z-index': items.length - item.n,
        position: 'absolute',
      }"
      >
      {{ item.n }}
    </div>
  </div>
</template>

<style scoped>
.window {
  background-color: green;
  color: white;
  box-shadow: 3px 3px 20px rgba(0, 0, 0, 0.5);
}
#bg {
  background-color: cyan;
  width: 1000px;
  height: 700px;
  position: relative;
}
</style>
