<template>
  <v-container>
    <v-row>
      <v-col class="mb-4">
        <h1 class="display-2 font-weight-bold mb-3">Товары</h1>
      </v-col>

      <v-col class="mb-5" cols="12">
        <v-data-table
          :headers="headers"
          :items="products"
          :loading="loading"
          :server-items-length="length"
          :options.sync="options"
          :footer-props="footer"
          :search="search"
          no-data-text="Товаров не найдено"
          class="elevation-1"
        >
          <template v-slot:top>
            <v-toolbar flat>
              <v-text-field
                v-model="search"
                label="Поиск по названию или артикулу"
                class="mx-4"
              ></v-text-field>
              <v-dialog v-model="dialog" max-width="500px">
                <template v-slot:activator="{ on, attrs }">
                  <v-btn color="primary" dark class="mb-2" v-bind="attrs" v-on="on">
                    Добавить товар
                  </v-btn>
                </template>
                <v-card>
                  <v-card-title>
                    <span class="text-h5"
                      >{{ editIndex >= 0 ? "Редактировать" : "Добавить" }} товар</span
                    >
                  </v-card-title>

                  <v-card-text>
                    <v-container>
                      <v-form ref="form">
                        <v-row>
                          <v-col cols="12">
                            <v-text-field
                              v-model="editItem.name"
                              :rules="rules.name"
                              label="Название"
                            ></v-text-field>
                          </v-col>
                          <v-col cols="12">
                            <v-text-field
                              v-model="editItem.vendorCode"
                              :rules="rules.vendorCode"
                              label="Артикул"
                            ></v-text-field>
                          </v-col>
                        </v-row>
                      </v-form>
                    </v-container>
                  </v-card-text>

                  <v-card-actions>
                    <v-spacer></v-spacer>
                    <v-btn color="blue darken-1" text @click="close"> Отмена </v-btn>
                    <v-btn color="blue darken-1" text @click="save"> Сохранить </v-btn>
                  </v-card-actions>
                </v-card>
              </v-dialog>
              <v-dialog v-model="dialogDelete" max-width="500px">
                <v-card>
                  <v-card-title class="text-h5">
                    Вы уверены что хотите удалить этот товар?
                  </v-card-title>
                  <v-card-actions>
                    <v-spacer></v-spacer>
                    <v-btn color="blue darken-1" text @click="closeDelete">
                      Отмена
                    </v-btn>
                    <v-btn color="blue darken-1" text @click="deleteItemConfirm">
                      ОК
                    </v-btn>
                    <v-spacer></v-spacer>
                  </v-card-actions>
                </v-card>
              </v-dialog>
            </v-toolbar>
          </template>
          <template v-slot:item.actions="{ item }">
            <v-icon small class="mr-2" @click="updateItem(item)"> mdi-pencil </v-icon>
            <v-icon small @click="deleteItem(item)"> mdi-delete </v-icon>
          </template>
          <template v-slot:no-data>
            <v-btn color="primary" @click="fetch"> Обновить </v-btn>
          </template>
        </v-data-table>
      </v-col>
    </v-row>
    <v-snackbar
      :timeout="-1"
      :value="true"
      :color="snackbar.color"
      v-if="snackbar"
      absolute
      bottom
      right
    >
      {{ snackbar.message }}
    </v-snackbar>
  </v-container>
</template>

<script>
const HOST = process.env.VUE_APP_API ?? 'http://localhost';
console.log(HOST);

export default {
  name: "Products",

  data: () => ({
    search: "",
    headers: [
      {
        text: "Название",
        align: "start",
        value: "name",
        sortable: false,
      },
      {
        text: "Артикул",
        value: "vendorCode",
        sortable: false,
      },
      {
        text: "Действия",
        value: "actions",
        sortable: false,
      },
    ],
    loading: true,
    products: [],
    options: {
      itemsPerPage: 30,
    },
    length: 0,
    source: null,
    timer: null,
    footer: {
      itemsPerPageOptions: [],
    },
    editIndex: -1,
    dialog: false,
    dialogDelete: false,
    editItem: {
      name: "",
      vendorCode: "",
    },
    rules: {},
    defaultItem: {
      name: "",
      vendorCode: "",
    },
    snackbar: null,
  }),
  watch: {
    options: {
      handler() {
        this.fetch();
      },
    },
    search: {
      handler() {
        this.debouncedFetch();
      },
    },
    dialog(val) {
      val || this.close();
    },
    dialogDelete(val) {
      val || this.closeDelete();
    },
  },
  methods: {
    setSnackbar(snackbar) {
      this.snackbar = null;
      this.snackbar = snackbar;
      setTimeout(() => (this.snackbar = null), 2000);
    },
    close() {
      this.dialog = false;
      this.rules = {};
      this.$nextTick(() => {
        this.editItem = Object.assign({}, this.defaultItem);
        this.editIndex = -1;
      });
    },
    closeDelete() {
      this.dialogDelete = false;
      this.$nextTick(() => {
        this.editItem = Object.assign({}, this.defaultItem);
        this.editIndex = -1;
      });
    },
    save() {
      this.updateAbort();
      this.rules = {};
      this.editIndex < 0 ? this.create() : this.update();
    },
    updateItem({ id, name, vendorCode }) {
      this.editIndex = id;
      this.editItem = { name, vendorCode };
      this.dialog = true;
    },
    deleteItem({ id }) {
      this.editIndex = id;
      this.dialogDelete = true;
    },
    deleteItemConfirm() {
      this.delete();
      this.closeDelete();
    },

    afterSave(data) {
      if (data["@type"] === "ConstraintViolationList") {
        console.error(data);
        this.setSnackbar({
          color: "red",
          message: "Проверьте правильность заполнения формы",
        });
        this.rules = data["violations"].reduce((prev, { propertyPath, message }) => {
          if (prev[propertyPath]) {
            prev[propertyPath].push(message);
          } else {
            prev[propertyPath] = [message];
          }
          return prev;
        }, {});
        this.$refs.form.validate();
        return;
      }
      if (data["@type"] === "Product") {
        this.setSnackbar({ color: "green", message: "Товар успешно сохранён" });
      }
      this.close();
      this.fetch();
    },
    handleError(e) {
      console.error(e);
      this.setSnackbar({
        color: "red",
        message: "Произошла ошибка при сохранении, пожалуйста попробуйте позже",
      });
    },
    create() {
      fetch(new URL(`${HOST}/products`), {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(this.editItem),
      })
        .then((resp) => resp.json())
        .then(this.afterSave)
        .catch(this.handleError);
    },
    update() {
      fetch(new URL(`${HOST}/products/${this.editIndex}`), {
        method: "PUT",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(this.editItem),
      })
        .then((resp) => resp.json())
        .then(this.afterSave)
        .catch(this.handleError);
    },
    delete() {
      fetch(new URL(`${HOST}/products/${this.editIndex}`), { method: "DELETE" })
        .then((resp) => {
          if (resp.status !== 204) {
            throw new Error('Error delete');
          }
          this.setSnackbar({ color: "green", message: "Товар успешно удалён" });
          this.fetch();
        })
        .catch((e) => {
          console.error(e);
          this.setSnackbar({
            color: "red",
            message: "Произошла ошибка при удалении товара, пожалуйста попробуйте позже",
          });
        });
    },

    debouncedFetch() {
      clearTimeout(this.timer);
      this.timer = setTimeout(() => this.fetch(), 500);
    },
    fetch() {
      this.loading = true;
      this.updateAbort();
      const url = new URL(`${HOST}/products`);
      url.search = new URLSearchParams({
        page: this.options.page,
        name: this.search,
        vendorCode: this.search,
      }).toString();

      return fetch(url, { signal: this.source.signal })
        .then((resp) => resp.json())
        .then((data) => {
          this.length = data["hydra:totalItems"];
          this.products = data["hydra:member"];
          this.source = null;
        })
        .catch((e) => {
          console.error(e);
          this.source = null;
        })
        .finally(() => {
          this.loading = false;
        });
    },
    updateAbort() {
      if (this.source) {
        this.source.abort("Operation canceled by the user.");
      }
      this.source = new AbortController();
    },
  },
};
</script>
