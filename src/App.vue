<template>
  <div class="app">
    <div class="left-panel">
      <h2>ID Card Generator</h2>
      <div class="inputs">
        <input v-model="name" placeholder="Name" />
        <input v-model="region" placeholder="Region" />
        <input v-model="role" placeholder="Role" />
      </div>

      <div class="upload-area" @drop="onDrop" @dragover.prevent @dragenter.prevent>
        <input type="file" accept="image/*" @change="onFileSelect" />
        <p>Drag & drop or click to upload profile picture</p>
      </div>

      <p class="hint">Tip: drag text and photo directly on the template to fine-tune alignment.</p>

      <div class="controls" v-if="hasProfile">
        <button @click="resizeProfile(1.2)">+ Resize</button>
        <button @click="resizeProfile(0.8)">- Resize</button>
        <button @click="rotateProfile(90)">Rotate 90°</button>
        <button @click="flipHorizontal">Flip H</button>
        <button @click="resetProfile">Reset</button>
      </div>
    </div>

    <div class="right-panel">
      <div id="preview" class="preview-container">
        <v-stage :config="stageConfig" ref="stage">
          <v-layer>
            <v-image :config="templateConfig" />
            <v-group :config="textGroupConfig" @dragend="onTextGroupDragEnd">
              <v-text :config="nameConfig" />
              <v-text :config="regionConfig" />
              <v-text :config="roleConfig" />
            </v-group>
            <v-group v-if="profileImage" :config="profileGroupConfig">
              <v-image :config="profileImageConfig" @dragend="onProfileDragEnd" />
            </v-group>
          </v-layer>
        </v-stage>
      </div>
      <button @click="downloadCard">Download PNG</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      name: '',
      region: '',
      role: '',
      profileImage: null,
      templateImg: new window.Image(),
      stageWidth: 1280,
      stageHeight: 720,
      exportPixelRatio: 1,
      textGroup: {
        x: 0,
        y: 0,
        rotation: 24,
        draggable: true
      },
      profileFrame: {
        x: 0,
        y: 0,
        width: 0,
        height: 0,
        rotation: 24
      },
      profileImageTransform: {
        x: 0,
        y: 0,
        width: 0,
        height: 0,
        rotation: 0,
        scaleX: 1,
        scaleY: 1
      }
    };
  },
  computed: {
    stageConfig() {
      return {
        width: this.stageWidth,
        height: this.stageHeight
      };
    },
    templateConfig() {
      return {
        image: this.templateImg,
        width: this.stageWidth,
        height: this.stageHeight
      };
    },
    textGroupConfig() {
      return {
        x: this.textGroup.x,
        y: this.textGroup.y,
        rotation: this.textGroup.rotation,
        draggable: true
      };
    },
    nameConfig() {
      return {
        x: 0,
        y: 0,
        text: this.name || 'NAME',
        fontSize: Math.max(16, Math.round(this.stageWidth * 0.018)),
        fontFamily: 'Georgia, serif',
        fontStyle: 'bold',
        fill: '#ffffff',
        width: 105,
        wrap: 'word'
      };
    },
    regionConfig() {
      const nameFontSize = Math.max(16, Math.round(this.stageWidth * 0.018));
      const nameText = this.name || 'NAME';
      
      const nameLines = this.calculateTextLines(nameText, nameFontSize, 'bold', 105);
      const nameHeight = nameLines * nameFontSize * 1.5;
      
      return {
        x: 0,
        y: nameHeight + 8,
        text: this.region || 'REGION',
        fontSize: Math.max(13, Math.round(this.stageWidth * 0.015)),
        fontFamily: 'Georgia, serif',
        fill: '#ffffff',
        width: 105,
        wrap: 'word'
      };
    },
    roleConfig() {
      const nameFontSize = Math.max(16, Math.round(this.stageWidth * 0.018));
      const nameText = this.name || 'NAME';
      
      const nameLines = this.calculateTextLines(nameText, nameFontSize, 'bold', 105);
      const nameHeight = nameLines * nameFontSize * 1.5;
      
      const regionFontSize = Math.max(13, Math.round(this.stageWidth * 0.015));
      const regionText = this.region || 'REGION';
      
      const regionLines = this.calculateTextLines(regionText, regionFontSize, 'normal', 105);
      const regionHeight = regionLines * regionFontSize * 1.5;
      
      return {
        x: 0,
        y: nameHeight + 4 + regionHeight + 4,
        text: this.role || 'ROLE',
        fontSize: Math.max(13, Math.round(this.stageWidth * 0.015)),
        fontFamily: 'Georgia, serif',
        fill: '#ffffff',
        width: 105,
        wrap: 'word'
      };
    },
    profileGroupConfig() {
      return {
        x: this.profileFrame.x,
        y: this.profileFrame.y,
        rotation: this.profileFrame.rotation,
        clipX: -this.profileFrame.width / 2,
        clipY: -this.profileFrame.height / 2,
        clipWidth: this.profileFrame.width,
        clipHeight: this.profileFrame.height
      };
    },
    profileImageConfig() {
      return {
        image: this.profileImage,
        x: this.profileImageTransform.x,
        y: this.profileImageTransform.y,
        width: this.profileImageTransform.width,
        height: this.profileImageTransform.height,
        rotation: this.profileImageTransform.rotation,
        scaleX: this.profileImageTransform.scaleX,
        scaleY: this.profileImageTransform.scaleY,
        offsetX: this.profileImageTransform.width / 2,
        offsetY: this.profileImageTransform.height / 2,
        draggable: true
      };
    },
    hasProfile() {
      return !!this.profileImage;
    }
  },
  mounted() {
    this.templateImg.crossOrigin = 'anonymous';
    this.templateImg.onload = () => {
      const maxPreviewWidth = 980;
      const maxPreviewHeight = Math.max(480, Math.floor(window.innerHeight * 0.72));
      const fitScale = Math.min(
        maxPreviewWidth / this.templateImg.width,
        maxPreviewHeight / this.templateImg.height,
        1
      );

      this.stageWidth = Math.round(this.templateImg.width * fitScale);
      this.stageHeight = Math.round(this.templateImg.height * fitScale);
      this.exportPixelRatio = this.templateImg.width / this.stageWidth;
      this.applyDefaultLayout();
      this.$forceUpdate();  
    };
    this.templateImg.src = '/template.png';
  },
  methods: {
    calculateTextLines(text, fontSize, fontWeight, maxWidth) {
      // Approximate character width based on font and weight
      // Using very conservative estimates to prevent overlap
      const charWidth = fontWeight === 'bold' ? fontSize * 0.75 : fontSize * 0.7;
      
      const words = text.split(/\s+/).filter(w => w.length > 0);
      if (words.length === 0) return 1;
      
      const lines = [];
      let currentLine = '';
      
      words.forEach(word => {
        // Check if single word is too long and needs character wrapping
        const wordWidth = word.length * charWidth;
        if (wordWidth > maxWidth) {
          // Word itself is too long, needs to be wrapped at character level
          if (currentLine) {
            lines.push(currentLine);
            currentLine = '';
          }
          // Calculate how many lines this word needs
          const charsPerLine = Math.floor(maxWidth / charWidth);
          const wordLines = Math.ceil(word.length / charsPerLine);
          for (let i = 0; i < wordLines; i++) {
            lines.push(word.substr(i * charsPerLine, charsPerLine));
          }
          return;
        }
        
        const testLine = currentLine ? currentLine + ' ' + word : word;
        const estimatedWidth = testLine.length * charWidth;
        
        if (estimatedWidth > maxWidth && currentLine !== '') {
          // Line would be too long, start new line
          lines.push(currentLine);
          currentLine = word;
        } else {
          currentLine = testLine;
        }
      });
      
      if (currentLine) {
        lines.push(currentLine);
      }
      
      return Math.max(lines.length, 1);
    },
    applyDefaultLayout() {
      const w = this.stageWidth;
      const h = this.stageHeight;
      const profileWidth = w * 0.092;
      const profileHeight = h * 0.202;

      Object.assign(this.textGroup, {
        x: w * 0.62,
        y: h * 0.38,
        rotation: 22
      });

      Object.assign(this.profileFrame, {
        x: w * 0.788,
        y: h * 0.448,
        width: profileWidth,
        height: profileHeight,
        rotation: 22
      });

      this.resetProfileImageTransform();
    },
    resetProfileImageTransform() {
      const frameWidth = this.profileFrame.width;
      const frameHeight = this.profileFrame.height;

      if (!this.profileImage || frameWidth === 0 || frameHeight === 0) {
        Object.assign(this.profileImageTransform, {
          x: 0,
          y: 0,
          width: frameWidth,
          height: frameHeight,
          rotation: 0,
          scaleX: 1,
          scaleY: 1
        });
        return;
      }

      const imageWidth = this.profileImage.naturalWidth || this.profileImage.width;
      const imageHeight = this.profileImage.naturalHeight || this.profileImage.height;
      const scale = Math.max(frameWidth / imageWidth, frameHeight / imageHeight);

      Object.assign(this.profileImageTransform, {
        x: 0,
        y: 0,
        width: Math.round(imageWidth * scale),
        height: Math.round(imageHeight * scale),
        rotation: 0,
        scaleX: 1,
        scaleY: 1
      });
    },
    onFileSelect(e) {
      const file = e.target.files[0];
      this.loadProfile(file);
    },
    onDrop(e) {
      e.preventDefault();
      const file = e.dataTransfer.files[0];
      this.loadProfile(file);
    },
    loadProfile(file) {
      if (!file) {
        return;
      }

      const reader = new FileReader();
      reader.onload = (event) => {
        const img = new window.Image();
        img.src = event.target.result;
        img.onload = () => {
          this.profileImage = img;
          this.resetProfileImageTransform();
          this.$forceUpdate();
        };
      };
      reader.readAsDataURL(file);
    },
    resizeProfile(factor) {
      this.profileImageTransform.width *= factor;
      this.profileImageTransform.height *= factor;
      this.$forceUpdate();
    },
    rotateProfile(angle) {
      this.profileImageTransform.rotation += angle;
      this.$forceUpdate();
    },
    flipHorizontal() {
      this.profileImageTransform.scaleX *= -1;
      this.$forceUpdate();
    },
    resetProfile() {
      this.applyDefaultLayout();
      this.$forceUpdate();
    },
    onTextGroupDragEnd(e) {
      this.textGroup.x = e.target.x();
      this.textGroup.y = e.target.y();
    },
    onProfileDragEnd(e) {
      this.profileImageTransform.x = e.target.x();
      this.profileImageTransform.y = e.target.y();
      this.$forceUpdate();
    },
    async downloadCard() {
      const stage = this.$refs.stage.getStage();
      const dataURL = stage.toDataURL({
        pixelRatio: Math.max(2, this.exportPixelRatio),
        mimeType: 'image/png'
      });
      const link = document.createElement('a');
      link.download = `id-card-${this.name || 'user'}.png`;
      link.href = dataURL;
      link.click();
    }
  }
};
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Playfair+Display:wght@700;900&family=Inter:wght@400;500;600;700&display=swap');

.app {
  display: flex;
  height: 100vh;
  font-family: 'Inter', 'Segoe UI', sans-serif;
}

.app h2 {
  font-family: 'Playfair Display', serif;
  font-size: 38px;
  font-weight: 900;
  letter-spacing: 1px;
  color: #f5e6d3;
  margin-bottom: 24px;
  text-shadow: 0 3px 15px rgba(0,0,0,0.4);
  background: linear-gradient(135deg, #f5e6d3 0%, #d4a574 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.left-panel {
  width: 35%;
  padding: 30px;
  background: linear-gradient(135deg, #4a3728 0%, #2d1e14 50%, #1a0f08 100%);
  color: white;
  overflow-y: auto;
}

.right-panel {
  width: 65%;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 30px;
  background: linear-gradient(135deg, #1a1a1a 0%, #2d2520 50%, #1f1f1f 100%);
}

.inputs input {
  display: block;
  width: 100%;
  margin-bottom: 20px;
  padding: 15px;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-family: inherit;
  font-weight: 500;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.upload-area {
  border: 3px dashed rgba(255,255,255,0.5);
  padding: 40px;
  text-align: center;
  margin: 20px 0;
  cursor: pointer;
  border-radius: 12px;
  transition: all 0.3s;
}

.upload-area:hover {
  background: rgba(255,255,255,0.1);
}

.hint {
  margin: 0 0 16px;
  color: rgba(255, 255, 255, 0.9);
  font-size: 14px;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.controls button {
  flex: 1;
  min-width: 80px;
  padding: 12px 8px;
  border: none;
  border-radius: 6px;
  background: rgba(255,255,255,0.2);
  color: white;
  cursor: pointer;
  font-family: inherit;
  font-weight: bold;
}

.controls button:hover {
  background: rgba(255,255,255,0.3);
}

.preview-container {
  border: 3px solid #3a3a3a;
  border-radius: 12px;
  overflow: auto;
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
  margin-bottom: 30px;
  width: 100%;
  max-width: 1000px;
  max-height: 75vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #1a1a1a;
}

.preview-container :deep(.konvajs-content) {
  margin: 0 auto;
}

.right-panel button {
  padding: 18px 40px;
  font-size: 18px;
  font-weight: bold;
  font-family: inherit;
  background: linear-gradient(135deg, #8b5e2b 0%, #5a3921 50%, #3b1f08 100%);
  color: white;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  box-shadow: 0 5px 15px rgba(59,31,8,0.5);
  transition: all 0.3s;
}

.right-panel button:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 25px rgba(139,94,43,0.6);
}

@media (max-width: 980px) {
  .app {
    flex-direction: column;
    height: auto;
    min-height: 100vh;
  }

  .left-panel,
  .right-panel {
    width: 100%;
  }

  .preview-container {
    max-height: 60vh;
  }
}
</style>
