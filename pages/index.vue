<template>
  <section>
    <div id="mainBodyText" class="container text-center">
      <h1>Blockchain Certificate Verification</h1>
      <h3 class="lead">
        <em
          >With NotoCert, your certificates becomes tamper-proof, and it is easy
          to issue, to share and verify certificate.</em
        >
      </h3>
      <div style="padding-top: 120px;">
        <b-button variant="primary" size="lg" @click="chooseFiles()"
          ><b-icon-cloud-upload></b-icon-cloud-upload>
          {{ file ? file.name : 'Verify Certificate' }}</b-button
        >
        <input
          id="fileUpload"
          ref="fileInput"
          type="file"
          hidden
          @change="handleFileUpload()"
        />
        <div id="scrollDownP">
          <nuxt-link to="/notarise">Notarise one for free now </nuxt-link>
          <!-- <a href="#feature" class="js-scroll-trigger text-white"
            >Read more below</a
          > -->
        </div>
      </div>
    </div>
    <client-only>
      <stack-modal
        :show="showYes || showNo"
        title="Verification Result"
        :save-button="saveButton"
        :cancel-button="{ visible: false }"
        :modal-class="{ [modalClass]: true }"
        @close="
          showYes = false
          showNo = false
        "
        @save="btnOK"
      >
        <div v-if="showYes" class="stackModal">
          <p>This certificate is a notarised true copy!</p>

          <div
            v-if="loadedRatio > 0 && loadedRatio < 1"
            style="background-color: green; color: white; text-align: center;"
            :style="{ width: loadedRatio * 100 + '%' }"
          >
            {{ Math.floor(loadedRatio * 100) }}%
          </div>
          <pdf
            :src="pdfURL"
            :page="currentPage"
            style="border: 1px solid black;"
            @num-pages="pageCount = $event"
            @page-loaded="currentPage = $event"
            @progress="loadedRatio = $event"
          ></pdf>
          <p style="text-align: center;">
            <b-button
              :disabled="currentPage == 1"
              variant="primary"
              style="float: left;"
              @click="currentPage = currentPage - 1"
              >Previous</b-button
            >
            Page:<input
              v-model.number="currentPage"
              style="width: 1em; text-align: right;"
            />
            / {{ pageCount }}
            <b-button
              :disabled="currentPage == pageCount"
              variant="primary"
              style="float: right;"
              @click="currentPage = currentPage + 1"
              >Next</b-button
            >
          </p>
        </div>
        <p v-else-if="showNo" class="stackModal">
          This certificate is not a notarised true copy!
        </p>
      </stack-modal>
    </client-only>
  </section>
</template>

<script>
/* eslint-disable no-console */
/* eslint-disable no-global-assign */
/* eslint-disable eqeqeq */
import { RepositoryFactoryHttp } from 'symbol-sdk'
// import { PDFDocument } from 'pdf-lib'

export default {
  data() {
    return {
      // PDF properties
      pdfURL: null,
      loadedRatio: 0,
      currentPage: 1,
      pageCount: null,
      // Endpoint of blockchain
      nodeUrl:
        'http://api-01.ap-southeast-1.testnet-0951-v1.symboldev.network:3000',
      // File
      file: null,
      fileHash: null,
      // stack modal UI
      showYes: false,
      showNo: false,
      modalClass: '',
      saveButton: {
        title: 'Done',
        visible: true,
        btnClass: { 'btn btn-primary': true },
      },
    }
  },
  watch: {
    currentPage() {
      if (parseInt(this.currentPage) > this.pageCount) {
        this.currentPage = this.pageCount
      } else if (parseInt(this.currentPage) < 1) {
        this.currentPage = 1
      }
    },
    file() {
      if (this.file) {
        this.pdfURL = URL.createObjectURL(this.$refs.fileInput.files[0])
      }
    },
    showYes() {
      if (this.showYes === false) {
        this.btnOK()
      }
    },
    showNo() {
      if (this.showNo === false) {
        this.btnOK()
      }
    },
  },
  methods: {
    btnOK() {
      this.showYes = false
      this.showNo = false
      this.file = null
      this.fileHash = null
    },
    chooseFiles() {
      // file explorer to browse + select file
      document.getElementById('fileUpload').click()
    },
    arrayBufferToWordArray(ab) {
      const i8a = new Uint8Array(ab)
      const a = []
      for (let i = 0; i < i8a.length; i += 4) {
        a.push(
          (i8a[i] << 24) | (i8a[i + 1] << 16) | (i8a[i + 2] << 8) | i8a[i + 3]
        )
      }
      return this.CryptoJS.lib.WordArray.create(a, i8a.length)
    },
    handleFileUpload() {
      // Trigger v-on:change when file is uploaded
      this.file = this.$refs.fileInput.files[0]
      if (this.file) {
        // Check if there is a file attached on change
        this.loader = this.$loading.show({
          // show loading overlay
          container: this.fullPage ? null : this.$refs.formContainer,
          canCancel: false,
          onCancel: this.onCancel,
          color: 'white',
          loader: 'bars',
          backgroundColor: '#000000',
          opacity: 0.7,
          zIndex: 100,
        })
        if (
          this.file.type === 'application/pdf' &&
          this.file.name.includes(' - ')
        ) {
          self = this
          const reader = new FileReader()
          reader.onload = function (event) {
            const data = new Uint8Array(event.target.result)
            self.fileHash = self.CryptoJS.SHA256(
              self.arrayBufferToWordArray(data)
            )
            // const pdfDoc = await PDFDocument.load(data)
            // const txHash = pdfDoc.getKeywords()
            let txHash = self.file.name.split(' - ')[
              self.file.name.split(' - ').length - 1
            ]
            txHash = txHash.replace('.pdf', '')
            self.verifyFileOnBlockchain(txHash)
          }
          reader.readAsArrayBuffer(this.file)
        } else if (this.file.type !== 'application/pdf') {
          this.loader.hide()
          this.file = null
          this.$bvToast.toast(`Please upload certificate in pdf format only!`, {
            title: 'Verify Certificate Notification',
            autoHideDelay: 6000,
            appendToast: false,
            variant: 'warning',
          })
        } else {
          this.loader.hide()
          this.file = null
          this.$bvToast.toast(
            `The uploaded file is probably modified as it does not match the format for verfication`,
            {
              title: 'Verify Certificate Notification',
              autoHideDelay: 6000,
              appendToast: false,
              variant: 'warning',
            }
          )
        }
      }
    },
    verifyFileOnBlockchain(txHash) {
      // testing function
      const operators1 = require('rxjs/operators')
      const repositoryFactory = new RepositoryFactoryHttp(this.nodeUrl)
      const transactionHttp = repositoryFactory.createTransactionRepository()

      transactionHttp
        .getTransaction(txHash)
        .pipe(operators1.map((x) => x))
        .subscribe(
          (transaction) => {
            // Reject it if the TX reciever is not from the Notarise account
            // Notarise account also have tx restriction feature in
            if (
              transaction.recipientAddress.address ===
              'TAUSJXSGAZSLAX2SYHRHWPAHOLTRYMFB5L3QVC5K'
            ) {
              this.loader.hide()
              if (transaction.message.payload == this.fileHash) {
                this.showYes = true
              } else {
                this.showNo = true
              }
            } else {
              this.$bvToast.toast(
                `The uploaded certificate is not notarised by NotoCert.`,
                {
                  title: 'Verify Certificate Notification',
                  autoHideDelay: 6000,
                  appendToast: false,
                  variant: 'danger',
                }
              )
            }
          },
          (err) => {
            // since the txHash on the file is modified/wrong/doesn't exist
            this.loader.hide()
            this.file = null
            this.$bvToast.toast(
              `The uploaded file is probably modified or doesn't exist in the blockchain`,
              {
                title: 'Verify Certificate Notification',
                autoHideDelay: 6000,
                appendToast: false,
                variant: 'warning',
              }
            )
            console.log(err)
          }
        )
    },
  },
  head() {
    return { title: 'Home - Verify Certificate' }
  },
}
</script>

<style scoped>
section {
  padding-top: 120px;
  width: 100%;
  height: 100vh;
  background-size: cover;
  background-repeat: no-repeat;
  background: linear-gradient(rgba(0, 0, 0, 0.2), rgba(0, 0, 0, 0.2)),
    url('~assets/img/header.jpg');
}
section #mainBodyText {
  font-family: Verdana, Geneva, Tahoma, sans-serif;
  color: white;
}
.stackModal {
  color: black;
}
#scrollDownP a {
  font-size: 11px;
  padding-top: 15px;
  color: gray;
  text-decoration: underline;
}
#scrollDownP a:hover {
  text-decoration: underline;
  color: white;
}
</style>
