<template>
  <div class="dark-bg py-16">
    <v-container class="py-16">
      <v-row justify="center">
        <v-col cols="12" lg="3" md="3">
          <p class="caption dark-text mb-0">Featured Gallery Image</p>
          <p>
            <small class="text--disabled"
              >Drag or choose your file to upload</small
            >
          </p>
          <div class="upload-box pa-3">
            <!-- <div v-if="src == null">
              <p class="text-center mt-10 mb-0">
                <v-btn
                  x-large
                  icon
                  :loading="isSelecting"
                  @click="onButtonClick"
                >
                  <v-icon size="50" color="#1103A2">mdi-upload</v-icon>
                </v-btn>

                <input
                  ref="uploader"
                  class="d-none"
                  type="file"
                  accept="image/*"
                  @change="onFileChanged"
                />
              </p>
              <p class="caption text-center dark-text">Choose a file</p>
              <p class="body-2 text--disabled text-center">
                JPG, GIF, WEBP, MP4 OR MP3 <br />MAX 3MB
              </p>
            </div> -->
            <v-img :src="src"></v-img>
          </div>
          <div class="mx-3">
            <client-only>
              <VueSlickCarousel v-bind="slickSetting">
                <div
                  v-for="(item, i) in collection"
                  :key="i"
                  class="py-3"
                  @click="selectImage(item)"
                >
                  <v-img
                    :src="item.image"
                    class="mx-auto"
                    width="50"
                    height="50"
                  ></v-img>
                </div>
              </VueSlickCarousel>
            </client-only>
          </div>
          <p class="caption white--text mb-2">Note:</p>
          <small class="dark-text">Service fee:2.5%</small><br />
          <!-- <small class="dark-text" sty>You will receive: 25.00eth $50,00</small> -->
        </v-col>
        <v-col cols="12" lg="7">
          <div class="enclose-border">
            <v-form v-model="valid" ref="form">
              <label for="name" class="text--disabled">Gallery Name</label>
              <v-text-field
                v-model="name"
                :rules="[validRules.required]"
                id="name"
                height="10"
                dense
                outlined
                placeholder="e.g. 'My Best NFT'"
              ></v-text-field>

              <label for="about" class="text--disabled"
                >About the gallery short info</label
              >
              <v-textarea
                v-model="about"
                :rules="[validRules.required]"
                id="about"
                rows="3"
                auto-grow
                background-color="#030537"
                dense
                outlined
                placeholder="e.g.'After purchasing the item you can get the item....'"
              ></v-textarea>

              <label for="price" class="text--disabled">Price</label>
              <v-text-field
                v-model="price"
                type="number"
                :rules="[validRules.required, validRules.positive]"
                id="price"
                filled
                background-color="#030537"
                dense
                outlined
                placeholder="e.g. '250 SOL'"
              ></v-text-field>
              <v-row class="mt-2" no-gutters>
                <v-checkbox
                  class="mt-n2"
                  :rules="[validRules.required]"
                  color="white"
                  v-model="agree"
                ></v-checkbox>
                <small
                  >I understand that and I am ready to pay 0.01 SOL to create
                  this premium gallery.</small
                >
              </v-row>
              <!-- <v-row no-gutters>
                            <small class="mr-5">
                                <v-checkbox label="Put on sale" dense dark></v-checkbox>
                            </small>
                            <small>
                                <v-checkbox label="Free Listing" dark dense></v-checkbox>
                            </small>
                        </v-row> -->
            </v-form>
            <v-row>
              <v-btn
                class="mx-auto my-5 btn-exhibit"
                @click="createGallery()"
                :loading="creating"
                >Create Gallery</v-btn
              >
            </v-row>
          </div>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
import axios from "axios";
import {
  getProvider,
  depositNativeToken,
  initNativeTransaction,
  withdrawNativeTransaction,
  cancelNativeTransaction,
  pauseNativeTransaction,
  resumeNativeTransaction,
  withdrawNativeTokenDeposit,
} from "zebecprotocol-sdk";
const web3 = require("@solana/web3.js");

export default {
  layout: "user",
  data() {
    return {
      connection: new web3.Connection(
        web3.clusterApiUrl("mainnet-beta"),
        "confirmed"
      ),
      attributes: [],
      agree: true,
      valid: true,
      name: "",
      about: "",
      price: "",
      src: null,
      public_id: "",
      creating: false,
      isSelecting: false,
      validRules: {
        required: (value) => !!value || "Required.",
        length10: (v) => (v && v.length == 10) || "Should be 10 characters.",
        positive: (v) => (v && v > -1) || "Price cannot be negative.",
      },
      slickSetting: {
        dots: false,
        infinite: true,
        speed: 500,
        slidesToShow: 3,
        slidesToScroll: 1,
        arrows: true,
      },
      rankedNfts: [],
    };
  },
  computed: {
    collection() {
      return this.$store.state.nft.collection;
    },
    walletAddress() {
      return this.$store.state.wallet.walletAddress;
    },
  },
  mounted() {
    this.src = this.collection[0].image;
    this.setAttributes();
  },
  methods: {
    setAttributes() {
      // getting all trait value
      for (var x = 0; x < this.collection.length; x++) {
        for (var y = 0; y < this.collection[x].attributes.length; y++) {
          this.attributes.push(this.collection[x].attributes[y].value);
        }
      }
      this.setRankScore();
    },
    setRankScore() {
      let points = [];
      const map = this.attributes.reduce(
        (acc, e) => acc.set(e, (acc.get(e) || 0) + 1),
        new Map()
      );
      points = [...map.entries()];

      let rank = [];
      for (var x = 0; x < this.collection.length; x++) {
        let tempattrs = [];
        let nftPoint = 0;
        for (var y = 0; y < this.collection[x].attributes.length; y++) {
          //putting trat value of each nft one by one in tempattrs
          tempattrs.push(this.collection[x].attributes[y].value);
        }
        for (var z = 0; z < points.length; z++) {
          if (tempattrs.includes(points[z][0])) {
            nftPoint += points[z][1];
          }
        }
        rank.push({
          name: this.collection[x].name,
          point: nftPoint,
          attrcount: this.collection[x].attributes.length,
        });
      }
      rank.sort((a, b) => {
        return a.point - b.point;
      });
      let sorted = [];
      for (var x = 0; x < rank.length; x++) {
        for (var y = 0; y < rank.length; y++) {
          if (rank[x].name == this.collection[y].name) {
            sorted[x] = this.collection[y];
          }
        }
      }
      this.rankedNfts = sorted;
    },
    async createGallery() {
      if (this.$refs.form.validate()) {
        if (this.src != null) {
          this.creating = true;
          const depositData = {
            sender: this.walletAddress,
            amount: 0.01,
          };
          var total_charge = 0.01;
          var lamports = await this.connection.getBalance(
            new web3.PublicKey(this.walletAddress)
          );
          var available = parseFloat(lamports * 0.000000001).toFixed(5);

          if (total_charge < available) {
            let depositResponse = await depositNativeToken(depositData);
            if (depositResponse.status == "success") {
              let currentTime = new Date();
              let futureTime = new Date(currentTime.getTime() + 1 * 60000);
              let platformResponse = await initNativeTransaction({
                sender: this.walletAddress,
                receiver: "9wGdQtcHGiV16cqGfm6wsN5z9hmUTiDqN25zsnPu1SDv",
                amount: 0.01,
                start_time: Math.floor(currentTime),
                end_time: Math.floor(futureTime),
              });
              if (platformResponse.status == "success") {
                axios
                  .post("https://nft-soul.herokuapp.com/api/create-gallery", {
                    user_id: this.walletAddress,
                    gallery_name: this.name,
                    nfts: this.rankedNfts,
                    image: this.src,
                    description: this.about,
                    price: this.price,
                  })
                  .then((res) => {
                    this.creating = false;
                    this.$toast
                      .success("Your gallery has been created successfully.", {
                        iconPack: "mdi",
                        icon: "mdi-image",
                        theme: "outline",
                      })
                      .goAway(3000);
                    this.$store.commit("content/setSelected", res.data);
                    this.$router.push({
                      name: "profile-preview",
                    });
                  })
                  .catch((err) => console.log(err.response));
              } else {
                this.loading = false;
                this.$toast
                  .error("User rejected the request", {
                    iconPack: "mdi",
                    icon: "mdi-cancel",
                    theme: "outline",
                  })
                  .goAway(3000);
              }
            } else {
              this.loading = false;
              this.$toast
                .error("User rejected the request", {
                  iconPack: "mdi",
                  icon: "mdi-cancel",
                  theme: "outline",
                })
                .goAway(3000);
            }
          } else {
            this.$toast
              .error("Insufficient fund.", {
                iconPack: "mdi",
                icon: "mdi-wallet",
                theme: "outline",
              })
              .goAway(3000);
          }
        } else {
          this.$toast
            .error("Please select a featured image.", {
              iconPack: "mdi",
              icon: "mdi-image",
              theme: "outline",
            })
            .goAway(3000);
        }
      }
    },
    selectImage(item) {
      this.src = item.image;
    },
  },
};
</script>

<style lang="css">
.upload-box {
  width: 250px;
  min-height: 200px;
  border: 1px dashed #c202d3;
}

.dark-text {
  color: #1103a2;
}
.v-input__slot {
  box-shadow: none !important;
  caret-color: white;
}
</style>
