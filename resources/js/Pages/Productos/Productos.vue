<script lang="ts" setup>
import { ref, reactive, onMounted } from "vue";
import axios from "axios";
import AppLayout from "@/Layouts/AppLayout.vue";
import ProductoTabla from "@/Components/Productos/ProductoTabla.vue";
import ModalRegistrar from "@/Components/Productos/ModalRegistrar.vue";
import { toast } from "vue3-toastify";
import "vue3-toastify/dist/index.css";

const showModal = ref(false);

const productos = ref<Array<{
    id: number;
    name: string;
    code: string;
    price: number;
    categoryPDV: string;
    category: string;
    attributes: Array<{
        attributeId: string;
        attributeValues: Array<{ id: string; name: string }>;
        referenceGlobal: string;
        referencesInternal: Record<string, string>;
        extraPrice: number;
    }>;
}>>([]);

const productoEditado = ref(null);

const isUploading = ref(false);
const categoriasPDV = ref([]);
const categorias = ref([]);
const subcategorias = reactive<{ [key: number]: any[] }>({});
const subcategoriasMap = ref<{ [key: number]: string }>({});
const atributos = ref([]);
const valores_atributos = ref([]);
const producto = reactive({
    subcategoria1: null,
    subcategoria2: null,
    subcategoria3: null,
    subcategoria4: null,
});
const isLoading = reactive({
    1: false,
    2: false,
    3: false,
    4: false,
});
const baseUrl = import.meta.env.VITE_APP_ODOO_URL;
const showLinksModal = ref(false);
const idProductoRegistrado = ref([]);

const cerrarLinkModal = (): void => {
    showLinksModal.value = false;
};

const traerCategoriasPDV = async () => {
    try {
        const categoriasPDVCacheadas = localStorage.getItem("categoriasPDV");
        if (categoriasPDVCacheadas) {
            categoriasPDV.value = JSON.parse(categoriasPDVCacheadas);
            return;
        }

        const respuesta = await axios.get(`/categoriaspdv/traer`);
        categoriasPDV.value = respuesta.data;

        localStorage.setItem("categoriasPDV", JSON.stringify(respuesta.data));
    } catch (error) {
        console.error("Error al traer las categorías del PDV:", error);
    }
};

const traerCategorias = async () => {
    try {
        const categoriasCacheadas = localStorage.getItem("categorias");
        if (categoriasCacheadas) {
            categorias.value = JSON.parse(categoriasCacheadas);
            return;
        }

        const respuesta = await axios.get(`/categorias/traer`);
        categorias.value = respuesta.data;

        localStorage.setItem("categorias", JSON.stringify(respuesta.data));
    } catch (error) {
        console.error("Error al traer las categorías:", error);
    }
};

const traerSubcategorias = async (id: number, level: number) => {
    const cacheKey = `subcategories_level_${level}_${id}`;
    isLoading[level] = true;

    const cachedData = localStorage.getItem(cacheKey);
    if (cachedData) {
        subcategorias[level] = JSON.parse(cachedData);
        subcategorias[level].forEach((subcat) => {
            subcategoriasMap.value[subcat.id] = subcat.name;
        });
        isLoading[level] = false;
        return;
    }

    try {
        const response = await axios.get(`/subcategorias/traer/${id}`);
        subcategorias[level] = response.data;

        localStorage.setItem(cacheKey, JSON.stringify(response.data));

        response.data.forEach((subcat) => {
            subcategoriasMap.value[subcat.id] = subcat.name;
        });
    } catch (error) {
        console.error(`Error cargando subcategorías para id ${id}:`, error);
    } finally {
        isLoading[level] = false;
    }
};

const alCambiarCategoriaPrincipal = async (idCategoria: number) => {
    if (idCategoria) {
        await traerSubcategorias(idCategoria, 1);

        producto.subcategoria1 = null;
        producto.subcategoria2 = null;
        producto.subcategoria3 = null;
        producto.subcategoria4 = null;
    }
};

const alCambiarSubcategoria = async (idPadre: number, nivel: number) => {
    if (idPadre) {
        await traerSubcategorias(idPadre, nivel + 1);

        if (nivel === 1) {
            producto.subcategoria2 = null;
            producto.subcategoria3 = null;
            producto.subcategoria4 = null;
        } else if (nivel === 2) {
            producto.subcategoria3 = null;
            producto.subcategoria4 = null;
        } else if (nivel === 3) {
            producto.subcategoria4 = null;
        }
    }
};

const traerAtributos = async () => {
    try {
        const atributosCacheadas = localStorage.getItem("atributos");
        if (atributosCacheadas) {
            atributos.value = JSON.parse(atributosCacheadas);
            return;
        }

        const respuesta = await axios.get(`/atributos/traer`);
        atributos.value = respuesta.data;

        localStorage.setItem("atributos", JSON.stringify(respuesta.data));
    } catch (error) {
        console.error("Error al traer los atributos:", error);
    }
};

const traerValoresAtributos = async (idAtributo: number) => {
    try {
        const cacheKey = `valores_atributos_${idAtributo}`;
        const cachedData = localStorage.getItem(cacheKey);

        if (cachedData) {
            valores_atributos.value[idAtributo] = JSON.parse(cachedData);
            return;
        }

        const response = await axios.get(`/valores_atributos/traer/${idAtributo}`);
        valores_atributos.value[idAtributo] = response.data;

        localStorage.setItem(cacheKey, JSON.stringify(response.data));
    } catch (error) {
        console.error("Error al traer valores de atributos:", error);
    }
};

const openModal = async (modo: "agregar" | "editar", productoSeleccionado: any = null) => {
    // console.log("Abrir Modal - Modo:", modo, "Producto seleccionado:", productoSeleccionado);

    if (modo === "agregar") {
        productoEditado.value = null;
    } else if (modo === "editar") {
        productoEditado.value = { ...productoSeleccionado };

        if (productoSeleccionado.category) {
            await traerSubcategorias(productoSeleccionado.category, 1);
        }
        if (productoSeleccionado.subcategory1) {
            await traerSubcategorias(productoSeleccionado.subcategory1, 2);
        }
        if (productoSeleccionado.subcategory2) {
            await traerSubcategorias(productoSeleccionado.subcategory2, 3);
        }
        if (productoSeleccionado.subcategory3) {
            await traerSubcategorias(productoSeleccionado.subcategory3, 4);
        }
    }
    showModal.value = true;
};


const closeModal = () => {
    showModal.value = false;
};

const agregarProducto = (producto: any) => {
    const nuevoProducto = {
        ...producto,
        category: String(producto.category || "Sin categoría"),
        attributes: producto.attributes || [],
    };
    nuevoProducto.id = productos.value.length + 1;
    productos.value.push(nuevoProducto);
    // console.log(productos.value);
    closeModal();
};

const actualizarProducto = (productoActualizado: any) => {
    const categoriaValida = categorias.value.find(
        (cat) => String(cat.id) === String(productoActualizado.category)
    );

    if (!categoriaValida) {
        toast.error("Categoría principal no válida.", {
            autoClose: 3000,
            position: "bottom-right",
        });
        return;
    }

    const index = productos.value.findIndex((producto) => producto.id === productoActualizado.id);
    if (index !== -1) {
        productos.value[index] = { ...productoActualizado };
    } else {
        toast.error("No se pudo actualizar el producto, no encontrado.", {
            autoClose: 3000,
            position: "bottom-right",
        });
    }
    closeModal();
};


const registrarProductos = async () => {
    isUploading.value = true;
    try {
        if (!productos.value.length) {
            toast.warn("No hay productos para registrar.", {
                autoClose: 3000,
                position: "bottom-right",
            });
            return;
        }

        const response = await axios.post('/productos/registrar_todo', productos.value);
        idProductoRegistrado.value = response.data.product_ids;
        showLinksModal.value = true;

        if (response.status === 200) {
            // console.log("Productos registrados con éxito:", response.data);
            toast.success("Productos registrados con éxito.", {
                autoClose: 3000,
                position: "bottom-right",
            });
            productos.value = [];
        } else {
            // console.error("Error al registrar productos:", response.data);
            toast.error("Ocurrió un error al registrar los productos.", {
                autoClose: 3000,
                position: "bottom-right",
            });
        }
    } catch (error) {
        console.error("Error al enviar los productos:", error);
        toast.error("Ocurrió un error al registrar los productos.", {
            autoClose: 3000,
            position: "bottom-right",
        });
    } finally {
        isUploading.value = false;
    }
};

const duplicarProducto = (producto: any) => {
    try {
        const newId = productos.value.length + 1;

        const duplicatedProducto = {
            ...producto,
            id: newId,
            name: `${producto.name} (Copia)`,
        };

        if (duplicatedProducto.attributes) {
            duplicatedProducto.attributes = duplicatedProducto.attributes.map((attr) => ({
                ...attr,
                attributeValues: [...attr.attributeValues],
                referencesInternal: { ...attr.referencesInternal },
            }));
        }

        productos.value.push(duplicatedProducto);

        toast.success("Producto duplicado con éxito.", {
            autoClose: 3000,
            position: "bottom-right",
        });
    } catch (error) {
        console.error("Error al duplicar el producto:", error);

        toast.error("Ocurrió un error al duplicar el producto.", {
            autoClose: 3000,
            position: "bottom-right",
        });
    }
};

const eliminarProducto = (id: number) => {
    const index = productos.value.findIndex((producto) => producto.id === id);
    if (index !== -1) {
        productos.value.splice(index, 1);
        toast.success("Producto eliminado con éxito.", {
            autoClose: 3000,
            position: "bottom-right",
        });
    } else {
        toast.error("No se pudo eliminar el producto.", {
            autoClose: 3000,
            position: "bottom-right",
        });
    }
};

onMounted(() => {
    traerCategoriasPDV();
    traerCategorias();
    traerAtributos();
});
</script>

<template>
    <AppLayout title="Productos">
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Productos
            </h2>
        </template>

        <div class="py-12">
            <div class="max-w-12xl mx-auto sm:px-6 lg:px-1">
                <div class="bg-white overflow-hidden shadow-xl sm:rounded-lg p-6">
                    <ProductoTabla @cerrarLinkModal="cerrarLinkModal" :idProductoRegistrado="idProductoRegistrado"
                        :baseUrl="baseUrl" :showLinksModal="showLinksModal" :isUploading="isUploading"
                        :productos="productos" :categoriasPDV="categoriasPDV" :categorias="categorias" :subcategorias-map="subcategoriasMap"
                        :atributos="atributos" @agregar="(producto) => openModal('agregar', producto)" @registrar="registrarProductos"
                        @duplicar="duplicarProducto" @eliminar="eliminarProducto" @editar="(producto) => openModal('editar', producto)" />
                </div>
            </div>
        </div>

        <ModalRegistrar :show="showModal" @close="closeModal" @submit="agregarProducto" @update="actualizarProducto"
            :productoInicial="productoEditado" :isLoading="isLoading" :categoriasPDV="categoriasPDV" :categorias="categorias"
            :subcategorias="subcategorias" :atributos="atributos" :valores_atributos="valores_atributos"
            @cambiarCategoriaPrincipal="alCambiarCategoriaPrincipal" @cambiarSubcategoria="alCambiarSubcategoria"
            @cambiarAtributo="traerValoresAtributos" />
    </AppLayout>
</template>
