<template>
  <div class="w-2/3 container mx-auto mt-32 ">
    <div class="flex ">  
        <div class="w-2/6">
            <carousel class="carousel " :perPage="1" >
            <slide v-for="item in images" :key="item"> <img :src="item" class="slide rounded-xl shadow-xl mx-5 mb-10" /></slide>
            </carousel>
        </div>
        <div class="bg-white w-4/6 px-10 py-2 rounded-xl shadow-lg">
            <div><i class="fas fa-times-circle text-red-500 text-2xl hover:text-red-700 block float-right cursor-pointer" @click="showConfirm=true" v-if="roll === 'admin'"></i>
              <i class="fas fa-edit text-yellow-500 text-2xl hover:text-yellow-700 cursor-pointer float-right mr-5" @click="updateProduct(product.uuid)" v-if="roll === 'admin'"></i>
            </div>

            <h2 class="font-bold text-2xl text-left">{{product.name}}</h2>
            <p class="text-left text-gray-500 mb-5 text-md"> {{ getCategory(product.category_uuid) }}</p>
            <p class="font-bold text-2xl text-left mb-5"> $ {{product.price}} MXM</p>
            <form class="flex mb-5">
                <div class="select mr-5">
                    <label class="text-left block text-gray-500">Talla:</label>
                  <v-select v-model="size" :options="sizesAvaible" />

                </div>
                <div  class="inputNumber">
                    <label class="text-left block text-gray-500">Cantidad:</label>
                    <vs-input v-model="amount" type="number" min="1"/>
                </div>

            </form>                
            <vs-button @click="submitted(product.uuid)" success class="block w-1/2 buttonAdd">Agregar al carrito</vs-button>
            <p for="descripcion" class="font-bold text-2xl text-left mt-10">Descripción:</p>
            <p class="text-left text-gray-500 mb-5 text-md">{{product.description}}</p>

            <p for="descripcion" class="font-bold text-2xl text-left mt-10">Cambios y devoluciones:</p>
            <p class="text-justify my-2 text-gray-500">Al hacer tu compra en línea tienes 30 días naturales a partir de que se genere tu pedido para solicitar un cambio.
                Por el momento no realizamos devoluciones. Para más información consulta nuestros términos y condiciones
            </p>
        </div>
    </div>

    <vs-dialog width="550px" not-center v-model="showConfirm">
        <template #header>
          <h4 class="not-margin">
            Eliminarás este producto
          </h4>
        </template>
        <div class="con-content">
          <p>
            ¿Estás seguro de que quieres eliminar este producto? No habrá recuperación de ello si aceptas. 
          </p>
        </div>

        <template #footer>
          <div class="con-footer">
            <vs-button @click="deleteProduct" success flat>
              Eliminar
            </vs-button>
            <vs-button @click="showConfirm=false" danger transparent>
              Cancelar
            </vs-button>
          </div>
        </template>
      </vs-dialog>
  </div>
</template>

<script>
import serviceAuth from "../services/auth";
import adminProducts from "../services/adminProducts";
import { Carousel, Slide } from "vue-carousel";
import config from "../services/config";
import {mapActions, mapState} from 'vuex'
import vSelect from 'vue-select'

export default {
  data() {
      return {
          config,
          product:{},
          images: [],
          categories: [],
          size: null,
          amount: 1,
          userData: {},
          showConfirm: false,
          sizesAvaible: [],
          stock:[]
      }
  },

  computed: {
    ...mapState(["token", "roll"]),
  },

  components: {
    Carousel,
    Slide,
    vSelect
  },

  created() {
    adminProducts.getProduct(this.$route.params.uuid).then((res) => {
      this.product = res.data.rows[0]
      this.getImages(this.product.uuid)
      this.sizesAvaible = this.product.size.split(',')
      this.theresStock();

    });

    adminProducts.getCategories().then((res) => {
        this.categories = res.data.rows;
    });

    if(this.token === null) {
      console.log('no logueado')
    } else {
      serviceAuth.dashboard().then(res => {
      this.userData = res.data.data.user;
      });
    }
  },

  methods: {
      ...mapActions(['settingTotalAmount']),


      getImages(uuid) {
          adminProducts.getImages(uuid).then((res) => {
            res.data.rows.map((item) => {
              this.images.push(item.url)
            })
          })
      },

      getCategory(uuid) {
        let category = ''
        this.categories.map((item) => {
            
            if(uuid === item.uuid) {
                category = item.category
            }
        })
        return category
      },

      submitted(product_uuid) {
        if(this.size === null ) {
          this.$vs.notification({
              icon: `<i class="fas fa-exclamation-circle"></i>`,
              color: 'danger',
              position: 'top-right',
              title: 'Ups!',
              text: `Selecciona una talla`,
          })
        } else {
             if(this.token === null) {
            const obj = {
              product_uuid: product_uuid,
              amount: parseInt(this.amount),
              size: this.size
            }
            let list = JSON.parse(localStorage.getItem('cartTemp'))
            list.push(obj)
            localStorage.setItem('cartTemp', JSON.stringify(list))
            this.$vs.notification({
              color: 'success',
              position: 'buttom-right',
              title:'Producto agregado',
              text: 'El producto ha sido agregado al carrito'
            })
            this.getCartPublic()
          } else {
            const obj = {
                product_uuid: product_uuid,
                user_uuid: this.userData.uuid,
                size: this.size,
                amount: this.amount
            }
            adminProducts.addToCart(obj).then((res) => {
              if(res.status === 200) {
                  this.$vs.notification({
                      color: 'success',
                      position: 'buttom-right',
                      title:'Producto agregado',
                      text: 'El producto ha sido agregado al carrito'
                  })
                  this.getCart()
              }
            })
          }
        }
      },

      getCart() {
        adminProducts.getCart(this.userData.uuid).then((res) => {
          let total = 0
          res.data.rows.map((item) => {
            total+=item.amount
          })
          this.settingTotalAmount(total)
        });
      },

      getCartPublic() {
        const list =JSON.parse(localStorage.getItem('cartTemp'))
        let total = 0
        list.map((item) => {
          total+= parseInt(item.amount,10)
        });
        this.settingTotalAmount(total)
    },

      deleteProduct() {
        this.showConfirm = false;

        adminProducts.deleteProduct(this.product.uuid).then((res) => {
          if(res.status === 200) {
            this.$vs.notification({
                icon: `<i class="fas fa-trash-alt"></i>`,
                color: 'success',
                position: 'top-center',
                title: 'Eliminado',
                text: `Ha sido eliminado el producto`
              })
            this.$router.push({name: 'Home'})
          }
        })
      },

      updateProduct(uuid) {
        this.$router.push({name: 'agregaProducto', params: { uuid: uuid }})
      },

      theresStock() {
        const my_split = this.product.stock.split('},')
            for(var i = 0; i<my_split.length; i++) {
                if(my_split.length > 1 && i !== my_split.length-1 ) {
                      this.stock.push(JSON.parse(my_split[i]+'}').stock)
                } else{
                      this.stock.push(JSON.parse(my_split[i]).stock)
                }
            }
        console.log(this.stock, 'Esta es mi estock')
      }

  }
};
</script>

<style>
.VueCarousel-pagination{
    margin-top: -4rem;
    background-color: transparent;
}


.slide{
    width: 80%;
    height: 400px;
    object-fit: cover;
}

.buttonAdd{
    width: 30%;
}

.inputNumber{
    width: 10%;
}

.select{
    width: 20%;
}
</style>