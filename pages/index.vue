<template>
  <div class="container">
    <div>
      <h3 class="title">
        失敗から知るAWSサーバレス <br />
        <small>~ API Gateway Lambda編 ~</small>
      </h3>
      <p class="subtitle">これは失敗から知るAWSサーバレスの動作検証用です。</p>
      <section class="upload-area">
        <input type="file" name="uploadImage" @change="changeFile" />
        <img :src="imageSrc" class="preview" />
      </section>
      <section class="result-area">
        <p class="size">
          raw image size: <strong>{{ rawImageSize }}</strong>
        </p>
        <p class="size">
          Bae64 size: <strong>{{ base64Size }}</strong>
        </p>
        <div v-show="uploadTarget">
          <button @click="postImage">
            画像をAPI Gateway x Lambda APIにPOSTする
          </button>
        </div>
        <textarea v-model="uploadTarget" class="result-base64" />
      </section>
      <section class="s3-images">
        <p class="title">S3イメージ</p>
        <ul>
          <li v-for="(image, index) in s3Images" :key="index">
            <div>
              <span>{{ image.Key }}</span>
              <button @click="deleteImage(image.Key)">削除</button>
            </div>
          </li>
        </ul>
      </section>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      uploadTarget: '',
      rawImageSize: '0 KB',
      base64Size: '0 KB',
      s3Images: []
    }
  },
  computed: {
    imageSrc() {
      return this.uploadTarget || '/images/NoImage.png'
    }
  },
  mounted() {
    this.getImages()
  },
  methods: {
    changeFile(data) {
      const {
        target: { files = [] }
      } = data
      if (!files.length) return
      const file = files[0]
      const reader = new FileReader()
      this.rawImageSize = this.getAppropriateSize(file.size)
      this.fileName = file.name.split('.')[0]
      reader.readAsDataURL(file)
      reader.onload = (evt) => {
        const size = evt.target.result.length
        this.base64Size = this.getAppropriateSize(size)
        this.uploadTarget = evt.target.result
      }
    },
    getAppropriateSize(size) {
      return size > 0
        ? size > 1048576
          ? `${(size / 1048576).toFixed(1)} MB`
          : `${(size / 1024).toFixed(1)} KB`
        : 0
    },
    /** S3にPOST後にGETで最新化する */
    async postImage() {
      await this.postS3Image()
      await this.getImages()
    },
    /** S3にDELETE後にGETで最新化する */
    async deleteImage(key) {
      await this.deleteS3Image(key)
      await this.getImages()
    },
    async getImages() {
      const { message: { Contents = [] } = {} } = await this.getS3Images()
      this.s3Images = Contents.filter((data) => data.Size > 0)
    },
    postS3Image() {
      return new Promise((resolve, reject) => {
        // あえてのXMLHttpRequest
        const xhr = new XMLHttpRequest()
        xhr.open('POST', `${process.env.BASE_URL}/images`, false)
        xhr.setRequestHeader('Content-Type', 'application/json')
        xhr.onload = (e) => {
          if (xhr.readyState === 4) {
            if (xhr.status === 200) {
              resolve(xhr.statusText)
            } else {
              reject(new Error(xhr.statusText))
            }
          }
        }
        xhr.onerror = (e) => {
          alert('500 Internal ServerError')
          reject(new Error(xhr.statusText))
        }
        xhr.send(`name=${this.fileName}&uploadImage=${this.uploadTarget}`)
        xhr.abort()
      })
    },
    getS3Images() {
      return new Promise((resolve, reject) => {
        // あえてのXMLHttpRequest
        const xhr = new XMLHttpRequest()
        xhr.open('GET', `${process.env.BASE_URL}/images`, false)
        xhr.setRequestHeader('Content-Type', 'application/json')
        xhr.onload = (e) => {
          if (xhr.readyState === 4) {
            if (xhr.status === 200) {
              resolve(JSON.parse(xhr.response))
            } else {
              reject(new Error(xhr.statusText))
            }
          }
        }
        xhr.onerror = (e) => reject(new Error(xhr.statusText))
        xhr.send(null)
      })
    },
    deleteS3Image(key) {
      console.log(key)
      return new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest()
        xhr.open('DELETE', `${process.env.BASE_URL}/images`, false)
        xhr.setRequestHeader('Content-Type', 'application/json')
        xhr.onload = (e) => {
          if (xhr.readyState === 4) {
            if (xhr.status === 200) {
              resolve(xhr.statusText)
            } else {
              reject(new Error(xhr.statusText))
            }
          }
        }
        xhr.onerror = (e) => reject(new Error(xhr.statusText))
        xhr.send(`key=${key}`)
      })
    }
  }
}
</script>

<style lang="scss">
html {
  font-size: 10px;
  .container {
    margin: 0 auto;
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    text-align: center;
  }

  .title {
    font-family: 'Quicksand', 'Source Sans Pro', -apple-system,
      BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial,
      sans-serif;
    display: block;
    font-weight: 300;
    font-size: 5rem;
    color: #35495e;
    letter-spacing: 1px;
  }

  .subtitle {
    font-weight: 300;
    font-size: 2rem;
    color: #526488;
    word-spacing: 5px;
    padding-bottom: 1.5rem;
  }

  section {
    margin: 2rem;
    &.upload-area {
      .preview {
        width: 25rem;
        height: 25rem;
      }
    }
    &.result-area {
      .size {
        font-size: 1.8rem;
        text-align: left;
        strong {
          color: red;
        }
      }
    }
    &.s3-images {
      .title {
        text-align: left;
      }
      ul {
        li {
          text-align: left;
          font-size: 1.8rem;
        }
      }
    }
  }

  .result-base64 {
    width: 100%;
    height: 25rem;
  }
}
</style>
