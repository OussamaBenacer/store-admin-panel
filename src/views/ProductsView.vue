<script setup>
import { ref, reactive } from "vue";
import {
  getProductsApi,
  addProductApi,
  updateProductApi,
  deleteProductApi,
} from "../services/products";
import { getCategoriesApi } from "@/services/categories";
import ConfirmDialog from "@/components/ConfirmDialog.vue";
import { watch } from "vue";
import { computed } from "vue";
import { onMounted } from "vue";

// State
const products = ref([]);
const categories = ref([]);
const categoryOptions = computed(() => [{ id: "", name: "All" }, ...categories.value]);
const loading = ref(false);

const filters = reactive({
  title: "",
  price_min: null,
  price_max: null,
  categoryId: "",
});

const page = ref(1);
const limit = 5;
const offset = computed(() => (page.value - 1) * limit);
const hasNextPage = computed(() => products.value.length == limit);
const hasPrevPage = computed(() => page.value !== 1);
let debounceTimeout = null;

watch(
  filters,
  () => {
    page.value = 1;
  },
  { deep: true },
);
watch(
  [offset, filters],
  ([offsetValue, filtersValue]) => {
    if (debounceTimeout) clearTimeout(debounceTimeout);

    debounceTimeout = setTimeout(() => {
      loadProducts(offsetValue, filtersValue);
    }, 400);
  },
  { immediate: true, deep: true },
);

async function loadProducts(offset, filters) {
  loading.value = true;
  try {
    const res = await getProductsApi(offset, limit, filters);
    console.log(res);
    const { data: productsData } = res;
    products.value = productsData;
  } catch (error) {
    console.error("Failed to load products:", error);
  } finally {
    loading.value = false;
  }
}

async function loadCategories() {
  loading.value = true;
  try {
    const { data: categoriesData } = await getCategoriesApi();
    categories.value = categoriesData;
  } catch (error) {
    console.error("Failed to load categories", error);
  } finally {
    loading.value = false;
  }
}

onMounted(loadCategories);

// Dialog & Form
const dialog = reactive({
  isOpen: false,
  isSending: false,
  editMode: false,
  selectedProduct: null,
  errors: [],
});

const initialForm = {
  title: "",
  price: 0,
  description: "",
  categoryId: "",
  images: "",
};

const form = ref(initialForm);
const formRef = ref(null);

const resetForm = () => {
  form.value = { ...initialForm };
  dialog.errors = [];
};

const openDialog = (product = null) => {
  dialog.editMode = !!product;
  dialog.selectedProduct = product ? { ...product } : null;

  form.value = {
    title: product?.title ?? "",
    price: product?.price ?? 0,
    description: product?.description ?? "",
    categoryId: product?.category?.id ?? "",
    images: product?.images?.join(" , ") ?? "",
  };

  dialog.isOpen = true;
};

const closeDialog = () => {
  dialog.isOpen = false;
  resetForm();
};

// Table headers
const headers = [
  { title: "Image", key: "images", sortable: false, align: "center" },
  { title: "Title", key: "title", sortable: false },
  { title: "Category", key: "category.name", sortable: false },
  { title: "Price", key: "price", sortable: false },
  { title: "Description", key: "description", sortable: false },
  { title: "Actions", key: "actions", sortable: false },
];

const saveProduct = async () => {
  const { valid } = await formRef.value.validate();

  if (!valid) return;

  dialog.isSending = true;
  dialog.errors = [];
  try {
    const finalProduct = {
      title: form.value.title,
      price: Number(form.value.price),
      description: form.value.description,
      categoryId: form.value.categoryId,
      images: form.value.images.split(",").map((url) => url.trim()),
    };

    if (dialog.editMode) {
      const changedFields = {};
      for (const key in finalProduct) {
        if (key === "images") {
          const newImages = finalProduct.images.join(",");
          const oldImages = (dialog.selectedProduct?.images || []).join(",");
          if (newImages !== oldImages) {
            changedFields.images = finalProduct.images;
          }
          continue;
        }

        if (finalProduct[key] != dialog.selectedProduct[key]) {
          changedFields[key] = finalProduct[key];
        }
      }
      await updateProductApi(dialog.selectedProduct?.id, changedFields);
    } else {
      await addProductApi(finalProduct);
    }

    await loadProducts(offset.value, filters);
    closeDialog();
  } catch (err) {
    console.error("Failed to save product:", err);
    const errors = err?.response?.data?.message || err?.message;
    dialog.errors = typeof errors === "string" ? [errors] : errors;
  } finally {
    dialog.isSending = false;
  }
};

// Delete product & delete dialog

const deleteDialog = reactive({
  isOpen: false,
  selectedProductId: null,
  isDeleting: false,
  errors: [],
});

const openDeleteDialog = (productId) => {
  deleteDialog.isOpen = true;
  deleteDialog.selectedProductId = productId;
};

const closeDeleteDialog = () => {
  deleteDialog.isOpen = false;
  deleteDialog.selectedProductId = null;
  deleteDialog.errors = [];
};

const deleteProduct = async () => {
  const productId = deleteDialog.selectedProductId;
  if (productId) {
    deleteDialog.isDeleting = true;
    deleteDialog.errors = [];

    try {
      await deleteProductApi(productId);
      closeDeleteDialog();
      await loadProducts(offset.value, filters);
    } catch (err) {
      const errorMsgs = err?.response?.data?.message || err?.message;
      deleteDialog.errors = typeof errorMsgs == "string" ? [errorMsgs] : errorMsgs;
    } finally {
      deleteDialog.isDeleting = false;
    }
  }
};
</script>

<!-- eslint-disable vue/valid-v-slot -->
<template>
  <v-card flat>
    <template v-slot:text>
      <div class="flex justify-between items-center">
        <h1 class="text-2xl font-semibold">Product Management</h1>
        <v-btn color="primary" @click="openDialog()">Add Product</v-btn>
      </div>
    </template>

    <v-card flat class="mb-4 p-4">
      <!-- filter part  -->
      <div class="flex flex-wrap items-center gap-4">
        <v-text-field v-model="filters.title" label="Title"></v-text-field>
        <v-text-field v-model="filters.price_min" label="Min Price" type="number"></v-text-field>
        <v-text-field v-model="filters.price_max" label="Max Price" type="number"></v-text-field>
        <v-select
          v-model="filters.categoryId"
          :items="categoryOptions"
          item-title="name"
          item-value="id"
          label="Category"
        >
        </v-select>
      </div>
    </v-card>

    <!-- table data part  -->
    <v-data-table
      title="Product management"
      :headers="headers"
      :items="products"
      :loading="loading"
      loading-text="Loading products..."
      :items-per-page="-1"
      hide-default-footer=""
    >
      <template v-slot:item.images="{ item }">
        <v-img
          aspect-ratio="1"
          :lazy-src="item.images[0]"
          :src="item.images[0]"
          class="rounded-md size-12 block mx-auto"
          cover
        ></v-img>
      </template>

      <template #item.actions="{ item }">
        <v-btn size="small" @click="openDialog(item)"> edit </v-btn>
        <v-btn size="small" color="error" @click="openDeleteDialog(item.id)"> delete </v-btn>
      </template>
    </v-data-table>

    <!-- pagination control  -->
    <div class="mb-10 px-4 py-3 flex items-center justify-center gap-4">
      <div>
        <v-btn :disabled="!hasPrevPage" @click="page--" class="mr-2"> Prev </v-btn>
        <v-btn :disabled="!hasNextPage" @click="page++"> Next </v-btn>
      </div>
      <p class="font-semibold">Page {{ page }} - products {{ products.length }} : {{ limit }}</p>
    </div>
  </v-card>

  <!-- Add / Edit Dialog -->
  <v-dialog v-model="dialog.isOpen" max-width="600">
    <v-form ref="formRef">
      <v-card :loading="dialog.isSending" :disabled="dialog.isSending">
        <v-card-title>
          <span class="text-h6">{{ dialog.editMode ? "Edit Product" : "Add Product" }}</span>
        </v-card-title>

        <v-card-text>
          <v-text-field
            v-model="form.title"
            label="Title"
            :rules="[(v) => !!v || 'Title is required']"
            validate-on="submit"
          />

          <v-text-field
            v-model="form.price"
            label="Price"
            type="number"
            :rules="[(v) => v > 0 || 'Price must be greater than 0']"
            validate-on="submit"
          />

          <v-textarea
            v-model="form.description"
            label="Description"
            :rules="[(v) => !!v || 'Description is required']"
            validate-on="submit"
          />

          <v-select
            v-model="form.categoryId"
            :items="categories"
            item-title="name"
            item-value="id"
            label="Category"
            :rules="[(v) => !!v || 'Category is required']"
            validate-on="submit"
          />

          <v-textarea
            v-model="form.images"
            label="Images (comma separated URLs)"
            :rules="[(v) => !!v || 'At least one image URL is required']"
            validate-on="submit"
          />

          <!-- Errors -->
          <ul v-if="dialog.errors?.length" class="text-red-500 text-sm mb-4">
            <li v-for="error in dialog.errors" :key="error">{{ error }}</li>
          </ul>
        </v-card-text>

        <v-card-actions>
          <v-btn variant="plain" @click="closeDialog">Cancel</v-btn>
          <v-btn color="primary" @click="saveProduct">Save</v-btn>
        </v-card-actions>
      </v-card>
    </v-form>
  </v-dialog>

  <ConfirmDialog
    v-model="deleteDialog.isOpen"
    title="Delete Product"
    content="Are you sure you want to delete this product?"
    confirm-text="delete"
    @confirm="deleteProduct"
    @discard="closeDeleteDialog"
    :is-sending="deleteDialog.isDeleting"
    color="error"
    :errors="deleteDialog.errors"
  />
</template>
