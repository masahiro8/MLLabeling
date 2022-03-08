<template>
  <div class="page">
    <div class="side">
      <div>
        <input @change="onChangeFile" multiple type="file" id="upload" />
      </div>
      <ul class="fileList">
        <li v-for="(file, index) in state.files" :key="index">
          <button
            @click="onSelectFile(index)"
            :class="isSelectedIndex(index) ? 'selected' : ''"
          >
            {{ file.name }}
          </button>
        </li>
      </ul>
    </div>
    <div class="main">
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
  </div>
</template>
<script>
import { ref, computed, reactive } from "vue";

const FormatImageProperty = {
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
};

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
      selectedIndex: null,
      files: [],
      practiveData: "",
      // クリックエリア
      isClickRect: {
        left: null,
        top: null,
        width: null,
        height: null,
      },
      //画像データ
      imageProperty: { ...FormatImageProperty },
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

    /**
     * ファイル一覧を追加
     */
    const setFiles = (files) => {
      if (files && files.length) {
        let _files = [];
        for (let i = 0; i < files.length; i++) {
          const format = { ...FormatImageProperty };
          format.name = files[i].name;
          format.path = URL.createObjectURL(files[i]);
          _files.push(format);
        }
        state.files = _files;
        console.log("==files", state.files);
      }
    };

    // リストからファイルを選択
    const onSelectFile = (index) => {
      //index取得
      state.selectedIndex = index;

      // キャンバスをクリア
      ClearCanvas();

      const file = { ...state.files[index] };

      // 画像情報更新
      state.imageProperty = file;

      // 画像ロード
      LoadImage({
        callback: () => {
          // キャンバスに描画
          DrawImageOnCanvas();
        },
      });
    };

    const isSelectedIndex = (index) => {
      return state.selectedIndex === index;
    };

    // ファイル変更
    const onChangeFile = (evt) => {
      if (evt.target.files.length) {
        // ファイル一覧
        setFiles(evt.target.files);
      }
    };

    // 画像をロード
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

    // キャンバスをクリア
    const ClearCanvas = () => {
      const ctx = root.value.getContext("2d");
      const rect = root.value.getBoundingClientRect();
      ctx.clearRect(0, 0, rect.width, rect.height);
    };

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
      state,
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
      onSelectFile,
      isSelectedIndex,
    };
  },
};
</script>
<style scoped>
.CanvasFrame {
  position: relative;
  border: 1px solid #eee;
}
.CanvasDraw {
  position: absolute;
  left: 0;
  top: 0;
}

.page {
  display: flex;
  height: 100%;
}

.side {
  width: 200px;
  height: 100%;
  overflow: auto;
  padding-right: 4px;
}

.main {
  flex: 1;
}
.fileList {
  padding: 0;
  margin: 0;
}

.fileList li {
  height: 32px;
  line-height: 32px;
  font-size: 16px;
  width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  display: flex;
  align-items: center;
  border-bottom: 1px solid #efefef;
}

.fileList li input[type="checkbox"] {
  padding: 2px;
  margin: 2px;
}

.fileList li button {
  border: none;
  background: none;
  width: 100%;
  height: 32px;
  text-align: left;
}

.fileList li button.selected {
  background-color: #efefef;
}
</style>
