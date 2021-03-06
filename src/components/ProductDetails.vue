<template>
  <div class="card text-center" v-on:click="toggleSale" :class="{ sale: onSale }">
    <div class="card-body">
      <h5 class="card-title">
        {{ product.name }}
      </h5>
      <h6 class="card-subtitle text-muted">
        SKU: {{ product.sku }}
      </h6>
      <div class="card-text">
        <img class="card-img-top img-thumbnail p-0 m-0" :src="imageLink" :alt="product.name"/>
      </div>
    </div>
    <div class="card-footer bg-white">
      <span class="sales-price">
        <span v-html="currencySymbol" />{{ product.salesPrice.amount | amount }}
      </span>
      <span class="list-price text-muted" v-if="product.listPrice">
        <span v-html="currencySymbol" />{{ product.listPrice.amount | amount }}
      </span>
    </div>
  </div>
</template>

<script>
/* eslint-disable */
import AccessTokenMixin from "@/mixins/AccessTokenMixin";
import axios from "axios";
import numeral from "numeral";
import uriTemplates from "uri-templates";
import _ from "lodash";

export default {
  name: "ProductDetails",

  mixins: [AccessTokenMixin],

  props: ["product", "reduction", "alerts"],

  computed: {
    onSale: function() {
      return this.product.tags && this.product.tags.includes("sale");
    },

    imageLink: function() {
      var link = _.get(this.product, "_links[default-image-data]", null);
      return link
        ? uriTemplates(link.href).fill({ width: 400, height: 200 })
        : "https://dummyimage.com/400x200/ffffff/0011ff.png&text=no+image";
    },

    // can't use a filter here, because of HTML escaping!
    currencySymbol: function() {
      switch (this.product.salesPrice.currency) {
        case "GBP":
          return "&pound";
        case "EUR":
          return "&euro;";
        default:
          return "???";
      }
    }
  },

  filters: {
    amount: function(value) {
      return numeral(value).format("0,0.00");
    }
  },

  methods: {
    toggleSale: function() {
      if (this.onSale) {
        this.removeFromSale();
      } else {
        this.putOnSale();
      }
    },

    /*
      1.) store current salesPrice as new listPrice
      2.) reduce salesPrice by x%
      3.) add sale tag
    */
    putOnSale: function() {
      // 1.) store current salesPrice as new listPrice
      const listPrice = {
        amount: this.product.salesPrice.amount,
        currency: this.product.salesPrice.currency,
        taxModel: this.product.salesPrice.taxModel
      };
      this.product.listPrice = listPrice;

      // 2.) reduce salesPrice by x%
      const salesPrice = {
        amount: listPrice.amount * (100 - this.reduction) / 100,
        currency: listPrice.currency,
        taxModel: listPrice.taxModel
      };
      this.product.salesPrice = salesPrice;

      // 3.) add sale tag
      this.product.tags = ["sale"];
    },

    /*
      1.) store current listPrice as new salesPrice
      2.) remove listPrice
      3.) remove sale tag
    */
    removeFromSale: function() {
      // 1.) store current listPrice as new salesPrice
      const salesPrice = {
        amount: this.product.listPrice.amount,
        currency: this.product.listPrice.currency,
        taxModel: this.product.listPrice.taxModel
      };
      this.product.salesPrice = salesPrice;

      // 2.) remove listPrice
      this.product.listPrice = null;

      // 3.) remove sale tag
      this.product.tags = [];
    },

    patchProduct: async function(patch) {
      if (this.isAccessTokenExpired()) {
        this.$router.push("/authorize");
      }

      const patchProduct = {
        baseURL: this.api_url,
        timeout: 5000,
        method: "PATCH",
        url: `/products/${this.product._id}`,
        headers: {
          "Content-Type": "application/json-patch+json",
          Authorization: `Bearer ${this.access_token.bearer}`
        },
        data: patch
      };

      axios
        .request(patchProduct)
        .then(response => {
          if (response.status === 200 && response.data) {
            this.product.salesPrice = response.data.salesPrice;
            this.product.listPrice = response.data.listPrice;
            this.product.tags = response.data.tags;
          } else {
            this.alerts.push({
              message: `error patching product ${this.product.sku}: ${
                request.statusText
              }`
            });
          }
        })
        .catch(e => {
          console.error(e);
          this.alerts.push({
            message: `error patching product ${this.product.sku}`
          });
        });
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.card {
  width: 17vw;
  height: 30vw;
  box-shadow: 2px 0px 10px #aaa;
  cursor: pointer;
}

.card-img-top {
  height: 80%;
}

.sale,
.sale .card-footer {
  background-color: lightpink !important;
}

.sale .sales-price {
  color: crimson;
}

.sale .list-price {
  text-decoration: line-through;
}
</style>
