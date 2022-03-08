<template>
  <div>
    <div>
      <input @change="onChangeFile" multiple type="file" id="upload" />
    </div>
    <div
      class="CanvasFrame"
      :style="computedCanvasFrameStyle"
      @click="onClickOnCanvas"
    >
      <div>
        <canvas ref="root" id="canvas"></canvas>
        <canvas
          ref="draw"
          class="CanvasDraw"
          :width="computedDrawWidth"
          :height="computedDrawHeight"
        ></canvas>
      </div>
    </div>
    <div>
      <textarea ref="rectValue" cols="50" rows="2"></textarea>
      <a :href="hrefTxt" target="_blank" :download="fileNameTxt">DL</a>
    </div>
  </div>
</template>
<script>
import { ref, computed, reactive } from "vue";
export default {
  name: "HelloWorld",

  setup() {
    // DOM
    const root = ref(null);
    // DOM
    const draw = ref(null);
    // DOM
    const rectValue = ref(null);

    const state = reactive({
      practiveData: "",
      // クリックエリア
      isClickRect: {
        left: null,
        top: null,
        width: null,
        height: null,
      },
      //画像データ
      imageProperty: {
        name: null,
        path: null,
        image: null,
        width: null,
        height: null,
        area: {
          top: null,
          left: null,
          width: null,
          height: null,
        },
        normarized: {
          top: null,
          left: null,
          right: null,
          bottom: null,
          width: null,
          height: null,
        },
      },
    });

    // キャンバスフレームのスタイル
    const computedCanvasFrameStyle = computed(() => {
      const { height, width } = state.imageProperty;
      return `width:${width}px;height:${height}px;`;
    });
    const computedDrawWidth = computed(() => {
      const { width } = state.imageProperty;
      return `${width}`;
    });
    const computedDrawHeight = computed(() => {
      const { height } = state.imageProperty;
      return `${height}`;
    });
    const hrefTxt = computed(() => {
      return state.imageProperty.practiveData;
    });
    const fileNameTxt = computed(() => {
      const filename = state.imageProperty.name;
      if (!filename) return "";
      const _filename = filename.substring(0, filename.lastIndexOf("."));
      return `${_filename}.txt`;
    });

    // キャンバス上をクリック
    const onClickOnCanvas = (evt) => {
      // 判定用 (リアクティブではない)
      const { top, left, width, height } = state.isClickRect;
      // グローバルの座標を取得
      const { x, y } = root.value.getBoundingClientRect();
      if (!top && !left) {
        // 1回目クリック
        state.isClickRect.top = evt.pageY - y;
        state.isClickRect.left = evt.pageX - x;
      } else {
        if (!width && !height) {
          // ２回目クリック
          state.isClickRect.height = evt.pageY - y - state.isClickRect.top;
          state.isClickRect.width = evt.pageX - x - state.isClickRect.left;

          // 保持
          state.imageProperty.area = { ...state.isClickRect };
          // 正規化
          const normalized = {
            top: state.isClickRect.top / state.imageProperty.height,
            left: state.isClickRect.left / state.imageProperty.width,
            right:
              (state.isClickRect.left + state.isClickRect.width) /
              state.imageProperty.width,
            bottom:
              (state.isClickRect.top + state.isClickRect.height) /
              state.imageProperty.height,
            width: state.isClickRect.width / state.imageProperty.width,
            height: state.isClickRect.height / state.imageProperty.height,
          };
          state.imageProperty.normarized = normalized;
          console.log("state.imageProperty", state.imageProperty);

          DrawRectOnCanvas();
          PutNormalizedData();
        } else {
          // 3回目クリック -> 1回目と同じ扱い
          state.isClickRect.top = evt.pageY - y;
          state.isClickRect.left = evt.pageX - x;
          state.isClickRect.height = null;
          state.isClickRect.width = null;
        }
      }
    };

    // ファイル変更
    const onChangeFile = (evt) => {
      if (evt.target.files.length) {
        // ファイルパスを取得
        const url = URL.createObjectURL(evt.target.files[0]);

        // 画像情報更新
        const _imageProperty = { ...state.imageProperty };
        _imageProperty.name = evt.target.files[0].name;
        _imageProperty.path = url;
        state.imageProperty = _imageProperty;

        // 画像ロード
        LoadImage({
          callback: () => {
            // キャンバスに描画
            DrawImageOnCanvas();
          },
        });
      }
    };

    const LoadImage = ({ callback }) => {
      const imagePath = state.imageProperty.path;
      const image = new Image();
      image.addEventListener("load", function () {
        if (root.value) {
          root.value.width = image.naturalWidth;
          root.value.height = image.naturalHeight;
          const _imageProperty = { ...state.imageProperty };
          _imageProperty.image = image;
          _imageProperty.width = image.naturalWidth;
          _imageProperty.height = image.naturalHeight;
          state.imageProperty = _imageProperty;
          console.log("load!");
          callback();
        }
      });
      image.src = imagePath;
    };

    // キャンバスに画像を描画
    const DrawImageOnCanvas = () => {
      const ctx = root.value.getContext("2d");
      ctx.drawImage(state.imageProperty.image, 0, 0);
    };

    // キャンバスに四角を描画
    const DrawRectOnCanvas = () => {
      console.log("area", { ...state.imageProperty.area });
      const { top, left, width, height } = state.isClickRect;
      const ctx = draw.value.getContext("2d");
      ctx.clearRect(
        0,
        0,
        state.imageProperty.width,
        state.imageProperty.height
      );
      ctx.beginPath();
      ctx.rect(left, top, width, height);
      ctx.strokeStyle = "red";
      ctx.lineWidth = 1;
      ctx.stroke();
    };

    //
    // [oject-class] [x_center] [y_center] [width] [height]
    const PutNormalizedData = () => {
      const _imageProperty = { ...state.imageProperty };
      const { top, left, width, height } = _imageProperty.normarized;
      const x_center = left + width / 2.0;
      const y_center = top + height / 2.0;
      const data = `${0} ${x_center} ${y_center} ${width} ${height}`;
      rectValue.value.value = data;

      // blob
      const blob = new Blob([data], { type: "text/plain" });
      state.imageProperty.practiveData = URL.createObjectURL(blob);
    };

    return {
      root,
      draw,
      rectValue,
      hrefTxt,
      fileNameTxt,
      computedCanvasFrameStyle,
      computedDrawWidth,
      computedDrawHeight,
      onChangeFile,
      onClickOnCanvas,
    };
  },
};
</script>
<style scoped>
.CanvasFrame {
  position: relative;
  border: 1px solid red;
}
.CanvasDraw {
  position: absolute;
  left: 0;
  top: 0;
  border: 1px solid blue;
}
</style>
