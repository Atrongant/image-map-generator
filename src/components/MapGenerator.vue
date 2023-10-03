<template>
  <div class="container">
    <div class="flex flex-row gap-4">
      <div>
        <label
          class="relative bg-sky-600 px-4 py-3 text-zinc-100 text-center flex justify-center items-center rounded-md"
          :for="fileId"
        >
          <input
            type="file"
            @change="handleFile"
            :id="fileId"
            name="file"
            ref="fileInput"
            accept="image/*"
            class="absolute opacity-0 overflow-hidden w-px h-px z-0"
          />
          <span>选择图片</span>
        </label>
      </div>
      <div>
        <div
          class="relative bg-green-600 px-4 py-3 text-zinc-100 text-center flex justify-center items-center rounded-md"
          @click="handleRemove(shape_selected)"
        >
          删除图形
        </div>
      </div>
    </div>
    <div class="flex justify-content-between">
      <div class="w-1/3 border flex justify-center items-center p-2">
        <div>
          <p class="text-center">图片区域</p>
          <div
            class="border-dashed border-round-md border-gray-500 p-2 border-2"
          >
            <div
              :style="{ width: width + 'px', height: height + 'px' }"
              class="relative"
            >
              <div :id="graphId" class="graph"></div>
              <img class="pure-img block" src ref="myImage" />
            </div>
          </div>
        </div>
      </div>
      <div class="w-1/3 border py-2">
        <p class="text-center">映射区域</p>
        <form>
          <div
            v-for="(item, key) in rects"
            :key="key"
            class="flex items-center justify-center gap-4 my-4"
          >
            <input
              type="text"
              placeholder="href"
              @keyup="generate_html"
              @focus="handleInputFocus($event, key)"
              :ref="'href-' + key"
            />
            <input
              type="text"
              placeholder="title"
              @keyup="generate_html"
              @focus="handleInputFocus($event, key)"
              :ref="'title-' + key"
            />
            <span
              class="bg-gray-400 px-2 py-1 text-zinc-100 text-center rounded-md text-sm"
              @click="handleRemove(key)"
            >
              删除图形
            </span>
          </div>
        </form>
      </div>
      <div class="w-1/3 border p-2">
        <p class="text-center">生成HTML代码</p>
        <form>
          <textarea
            class="w-full"
            placeholder="HTML will appear here!"
            rows="14"
            v-model="resultHtml"
            readonly
          ></textarea>
        </form>
      </div>
    </div>
  </div>
</template>
<script setup>
import {
  ref,
  reactive,
  toRefs,
  onMounted,
  defineProps,
  computed,
  getCurrentInstance,
} from "vue";
const { proxy } = getCurrentInstance();
const props = defineProps({
  index: {
    type: String || Number, // 属性类型
    required: true, // 是否必需
  },
});
SVG.Rect.prototype.aid = 0;
const data = reactive({
  filename: "",
  width: 396,
  height: 566,
  shape_selected: -1,
  in_shape: false,
  rects: [],
  rectsTmp: [],
  set_shape: "rect",
  r: null,
  resultHtml: "",
  clientX: -1,
  clientY: -1,
  drawing: {},
  image: {},
});
const { width, height, rects, resultHtml, shape_selected } = toRefs(data);
const fileInput = ref(null);
const myImage = ref(null);
// let drawing = {};
// let image = {};
onMounted(() => {});
const graphId = computed(() => {
  return `graph-${props.index}`;
});

const fileId = computed(() => {
  return `fileId-${props.index}`;
});
const imageId = computed(() => {
  return `mapped_image-${props.index}`;
});
function setSvgContainer() {
  data.drawing = new SVG(graphId.value);
}

function initSVG() {
  data.drawing.on("mousedown", function (event) {
    data.clientX = event.clientX;
    data.clientY = event.clientY;
    deselect_shapes();
    if (data.in_shape) return false;
    data.rects.push(data.drawing.rect());
    data.rectsTmp = new Array(data.rects.length).fill(false);
    data.in_shape = true;
    let rect = data.rects[data.rects.length - 1];
    rect.aid = data.rects.length - 1;
    rect.draw(event).fill("#333").opacity(".3");
    rect.draggable().on("dragmove", function () {
      generate_html();
    });
    rect.on("drawstop", function () {});
    rect.on(
      "mousedown",
      function () {
        deselect_shapes();
        this.selectize(true, {
          deepSelect: false,
          rotationPoint: false,
        }).resize();
        data.shape_selected = this.aid;
      },
      false
    );
  });
  data.drawing.on(
    "mouseup",
    function (event) {
      let r = data.rects[data.rects.length - 1];
      r.draw("stop", event);
      data.in_shape = false;
      if (data.clientX != event.clientX && data.clientY != event.clientY) {
        data.rectsTmp[data.rects.length - 1] = true;
        generate_html();
      } else {
        if (!data.rectsTmp[data.rects.lengt - 1])
          data.rects.splice(data.rects.length - 1);
      }
    },
    false
  );
}

function handleRemove(shape_selected) {
  if (shape_selected > -1) {
    data.shape_selected = shape_selected;
    let r = data.rects[data.shape_selected];
    r.selectize(!1);
    r.remove();
    data.rects.splice(data.shape_selected, 1);
    data.shape_selected = -1;
  }
}
function handleInputFocus(event, aid) {
  deselect_shapes();
  data.rects[aid].selectize(true);
  data.rects[aid]
    .selectize(true, {
      deepSelect: true,
      rotationPoint: false,
    })
    .resize();
  data.shape_selected = aid;
}
function handleAlert() {}

// 加载图片
function handleFile() {
  let file = fileInput.value.files[0];
  if (file) {
    data.image = new Image();
    setSvgContainer();
    data.image.src = createObjectURL(file);
    data.image.onload = () => {
      data.drawing.node.setAttribute("viewBox", "0 0 " + 396 + " " + 566);
      initSVG();
    };
    data.filename = file.name;
    myImage.value.src = data.image.src;
    myImage.value.width = data.width;
    myImage.value.height = data.height;
  }
  generate_html();
}
function createObjectURL(a) {
  return window.URL
    ? window.URL.createObjectURL(a)
    : window.webkitURL.createObjectURL(a);
}
function generate_html() {
  var a = '<img src="' + data.filename + '" usemap="#image_map">\n';
  a += '<map name="image_map">\n';
  for (let i = 0; i < data.rects.length; i++) {
    let r = data.rects[i];
    let type = r.type;
    let start = Math.round(r.attr("x")) + "," + Math.round(r.attr("y"));
    let end =
      Math.round(r.attr("x") + r.width()) +
      "," +
      Math.round(r.attr("y") + r.height());
    let coords = start + "," + end;
    var b = proxy.$refs["href-" + r.aid][0].value;
    var c = proxy.$refs["title-" + r.aid][0].value;
    a +=
      '  <area alt="' +
      c +
      '" title="' +
      c +
      '" href="' +
      b +
      '" coords="' +
      coords +
      '" shape="' +
      type +
      '">\n';
  }
  a += "</map>\n";
  data.resultHtml = a;
}
function deselect_shapes() {
  if (-1 < data.shape_selected) {
    for (let i = 0; i < data.rects.length; i++) data.rects[i].selectize(false);
    data.shape_selected = -1;
  }
}
</script>
<style>
.svg_select_points_lt {
  cursor: nw-resize;
}
.svg_select_points_rt {
  cursor: ne-resize;
}
.svg_select_points_rb {
  cursor: se-resize;
}
.svg_select_points_lb {
  cursor: sw-resize;
}
.svg_select_points_t {
  cursor: n-resize;
}
.svg_select_points_r {
  cursor: e-resize;
}
.svg_select_points_b {
  cursor: s-resize;
}
.svg_select_points_l {
  cursor: w-resize;
}
.svg_select_points_rot {
  stroke-width: 1;
  stroke: #000;
  fill: #f9ffed;
}
.svg_select_points_point {
  cursor: move;
}
.svg_select_boundingRect {
  stroke-width: 1;
  fill: gray;
  stroke-dasharray: 10 10;
  stroke: #000;
  stroke-opacity: 0.8;
  fill-opacity: 0.1;
  pointer-events: none;
}

.svg_select_points {
  fill: #fff;
  opacity: 0.8;
}
div[id^="graph-"] > svg {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
}
</style>
